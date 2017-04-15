## 背景

registry v2提供了新的基于token的鉴权方式，可以细粒度的配置不同用户对于不同镜像的权限，做到更好的镜像隔离。而公司正好需要使用鉴权，于是我们选用了开源项目docker auth。开始的时候使用都很正常，但是自从给私服配置了证书，推送镜像和拉取镜像都走https之后，镜像的推送经常会出现401错误（access to the requested resource is not authorized）。该问题和github上的一个issue很类似，

https://github.com/docker/docker/issues/17104(issue中提到的解决方案也是同事添加上去的)

写这篇文章的目的也就是给大家介绍下为什么会出现这个问题，同时避免大家再踩这个坑。。

## 线上环境

在介绍问题的根源之前，首先要介绍一下线上的环境。因为这个问题只有在特定的组网情况下才会出现。

为了更具有通用性，这里将组网大致概括为如下情况：用户请求通过前端nginx（开启443端口）转发到业务的nginx，然后由业务的nginx转发到不同的服务上。

![http://ww1.sinaimg.cn/large/87f5e2f6jw1fa3hitrdo0j20i80c4jrl.jpg](http://ww1.sinaimg.cn/large/87f5e2f6jw1fa3hitrdo0j20i80c4jrl.jpg)

虚线中的服务器上部署了nginx，同时也部署了registry v2以及其他的服务。这些服务（包括nginx）都采用docker部署，只有nginx对外暴露了80端口。

## 问题定位

当推送镜像的时候出现401，通过抓包发现当推送一个镜像层的时候PATCH的请求出现401导致整个镜像推送失败。经过定位发现PATCH请求发送到registry的时候没有携带对应的鉴权信息（添加一个名叫Authorization的header）。

通过阅读代码发现，当发送PATCH请求的时候会判断当前的url和整次推送请求的协议是否相同（整次请求的协议也就是这个镜像推送的时候是采用http还是https），如果不相同则不会添加对应的鉴权信息。这PATCH的请求是上一次发送的POST请求返回的Location，也就是说这个location是http的而不是https。

```

// /vendor/src/github.com/docker/distribution/registry/client/transport/transport.go

// tries to refresh/fetch a new token.

func (t *transport) RoundTrip(req *http.Request) (*http.Response, error) {

  req2 := cloneRequest(req)

  for _, modifier := range t.Modifiers {

    if err := modifier.ModifyRequest(req2); err != nil { // 这里进行修改

      return nil, err

    }

  }

  t.setModReq(req, req2)

  res, err := t.base().RoundTrip(req2)

  if err != nil {

    t.setModReq(req, nil)

    return nil, err

  }

  res.Body = &amp;onEOFReader{

    rc: res.Body,

    fn: func() { t.setModReq(req, nil) },

  }

  return res, nil

}



// /vendor/src/github.com/docker/distribution/registry/client/auth/session.go

func (ea *endpointAuthorizer) ModifyRequest(req *http.Request) error {

  v2Root := strings.Index(req.URL.Path, "/v2/")

  if v2Root == -1 {

    return nil

  }

  ping := url.URL{

    Host:   req.URL.Host,

    Scheme: req.URL.Scheme,

    Path:   req.URL.Path[:v2Root+4],

  }

  pingEndpoint := ping.String()

  challenges, err := ea.challenges.GetChallenges(pingEndpoint)

  if err != nil {

    return err

  }

  if len(challenges) &gt; 0 {

    for _, handler := range ea.handlers {

      for _, challenge := range challenges {

        if challenge.Scheme != handler.Scheme() // ##1 {

          continue

        }

        if err := handler.AuthorizeRequest(req, challenge.Parameters); err != nil {

          return err

        }

      }

    }

  }

  return nil

}

```

继续阅读registry的代码发现POST请求构建返回的location的时候是根据POST请求中的一个header(X-Forwarded-Proto)来判断当前的请求是http还是https，如果是http的话则设置Location为http协议(下面代码##1的位置)。

```

// /vendor/src/github.com/docker/distribution/registry/api/v2/urls.go

// NewURLBuilderFromRequest uses information from an *http.Request to

// construct the root url.

func NewURLBuilderFromRequest(r *http.Request) *URLBuilder {

  var scheme string

    // ##1

  forwardedProto := r.Header.Get("X-Forwarded-Proto")

    // 

  switch {

  case len(forwardedProto) &gt; 0:

    scheme = forwardedProto

  case r.TLS != nil:

    scheme = "https"

  case len(r.URL.Scheme) &gt; 0:

    scheme = r.URL.Scheme

  default:

    scheme = "http"

  }



  host := r.Host

  forwardedHost := r.Header.Get("X-Forwarded-Host")

  if len(forwardedHost) &gt; 0 {

    // According to the Apache mod_proxy docs, X-Forwarded-Host can be a

    // comma-separated list of hosts, to which each proxy appends the

    // requested host. We want to grab the first from this comma-separated

    // list.

    hosts := strings.SplitN(forwardedHost, ",", 2)

    host = strings.TrimSpace(hosts[0])

  }



  basePath := routeDescriptorsMap[RouteNameBase].Path



  requestPath := r.URL.Path

  index := strings.Index(requestPath, basePath)



  u := &amp;url.URL{

    Scheme: scheme,

    Host:   host,

  }



  if index &gt; 0 {

    // N.B. index+1 is important because we want to include the trailing /

    u.Path = requestPath[0 : index+1]

  }



  return NewURLBuilder(u)

}

```

这个header的作用是告诉服务器经过代理服务器转发的请求的原始协议是什么。

> a de facto standard for identifying the originating protocol of an HTTP request, since a reverse proxy (or a load balancer) may communicate with a web server using HTTP even if the request to the reverse proxy is HTTPS. An alternative form of the header (X-ProxyUser-Ip) is used by Google clients talking to Google servers.(https://en.wikipedia.org/wiki/List_of_HTTP_header_fields)



也就是说registry收到的X-Forwarded-Proto不是TLS。回顾下之前的线上环境，一个请求在到达registry之前经过了两次nginx转发。第一次转发将https变为了http，我们很容易就想到了是不是第一次转发的时候没有设置这个header，经过查证的确是这样。。

## 问题解决

最前端nginx转发的时候配置如下，

```

proxy_set_header  X-Forwarded-Proto $scheme;

```

