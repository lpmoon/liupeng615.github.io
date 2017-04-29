最近在为公司的docker registry搭建ui～在查找资料的时候发现docker registry为开发者预留一个扩展点 <em><strong>notifications。</strong></em>

该扩展点允许将registry上发生的事务(event)根据约定好的格式推送到特定的服务中去，这样就为registry的管理员提供了监控registry运行情况，以及图形化展示registry负载高低的可能。首先来看看notifications的架构图（摘自官网），

![http://ww3.sinaimg.cn/large/87f5e2f6jw1fa3fixc94fj20dh0g83za.jpg](http://ww3.sinaimg.cn/large/87f5e2f6jw1fa3fixc94fj20dh0g83za.jpg)

可以看到registry允许在notifications中配置多个endpoint。那么什么是endpoint呢？通俗点讲就是你自己实现的一个http服务，这个服务可以接受event，然后进行处理。这几个endpoint拥有相同的地位，也就是说同一个event会被下发多次。

通过上面的介绍大家或许对notifications的工作机制已经有了一些认识了，那么我们怎么在生产环境中使用呢？先来看一下使用notifications的配置，

```

notifications:

  endpoints:

    - name: alistener

      disabled: false

      url: https://my.listener.com/event

      headers: &lt;http.Header&gt;

      timeout: 500

      threshold: 5

      backoff: 1000

```

这里简单介绍下几个参数，disabled表示是否开启这个扩展点，url代表你自定义的服务，也就是用于接收event的。具体的解释请参考，<a href="http://docs.docker.com/registry/configuration/">configuration解释</a>。

下一步就是将这些参数使用起来了，我在官网上找了很久，找到了使用docker-compose启动registry的方法，但是docker-compose的配置文件中并不包含这些非同用参数，也就是说我们需要使用额外的方式进行传递。在查看了官方docker registry的docker file后终于找到了解决方法，

![http://ww1.sinaimg.cn/large/87f5e2f6jw1fa3fix3892j20is053tad.jpg](http://ww1.sinaimg.cn/large/87f5e2f6jw1fa3fix3892j20is053tad.jpg)

```

notifications:

    endpoints:

        - name: test 

          url: http://192.168.1.102:1234/

          headers:

             Authorization: [Bearer &lt;an example token&gt;]

          timeout: 1s

          threshold: 10

          backoff: 1s

          disabled: false

```

然后在docker-compose文件中写入如下配置，

```

registry:

  restart: always

  image: registry:2

  ports:

    - 5000:5000

  volumes:

    - /path/data:/var/lib/registry

    - /home/lpmoon/Docker/registry-compose:/etc/reg

  command:

    /etc/reg/config-dev.yml

```

上面做的主要工作是将/home/lpmoon/Docker/registry-compose目录（上面的config-dev.yml放置在这里）挂在到docker容器的/etc/reg目录上，同时设置启动的时候调用/etc/reg/config-dev.yml作为配置。

到此为止registry的所有工作已经就位，我们来实现我们自己的服务吧，由于docker-registry是go写的，为了能够直接引用里面的数据结构，我们也采用go来实现吧（～～学了一段时间的go，终于能用上了）。我们先来看看下面会用的一些基本的数据结构，

```

const (

  EventActionPull   = "pull"

  EventActionPush   = "push"

  EventActionDelete = "delete"

)



const (

  // EventsMediaType is the mediatype for the json event envelope. If the

  // Event, ActorRecord, SourceRecord or Envelope structs change, the version

  // number should be incremented.

  EventsMediaType = "application/vnd.docker.distribution.events.v1+json"

  // LayerMediaType is the media type for image rootfs diffs (aka "layers")

  // used by Docker. We don't expect this to change for quite a while.

  layerMediaType = "application/vnd.docker.container.image.rootfs.diff+x-gtar"

)



// Envelope defines the fields of a json event envelope message that can hold

// one or more events.

type Envelope struct {

  // Events make up the contents of the envelope. Events present in a single

  // envelope are not necessarily related.

  Events []Event `json:"events,omitempty"`

}



// TODO(stevvooe): The event type should be separate from the json format. It

// should be defined as an interface. Leaving as is for now since we don't

// need that at this time. If we make this change, the struct below would be

// called "EventRecord".



// Event provides the fields required to describe a registry event.

type Event struct {

  // ID provides a unique identifier for the event.

  ID string `json:"id,omitempty"`



  // Timestamp is the time at which the event occurred.

  Timestamp time.Time `json:"timestamp,omitempty"`



  // Action indicates what action encompasses the provided event.

  Action string `json:"action,omitempty"`



  // Target uniquely describes the target of the event.

  Target struct {

    // TODO(stevvooe): Use http.DetectContentType for layers, maybe.



    distribution.Descriptor



    // Length in bytes of content. Same as Size field in Descriptor.

    // Provided for backwards compatibility.

    Length int64 `json:"length,omitempty"`



    // Repository identifies the named repository.

    Repository string `json:"repository,omitempty"`



    // URL provides a direct link to the content.

    URL string `json:"url,omitempty"`

  } `json:"target,omitempty"`



  // Request covers the request that generated the event.

  Request RequestRecord `json:"request,omitempty"`



  // Actor specifies the agent that initiated the event. For most

  // situations, this could be from the authorizaton context of the request.

  Actor ActorRecord `json:"actor,omitempty"`



  // Source identifies the registry node that generated the event. Put

  // differently, while the actor "initiates" the event, the source

  // "generates" it.

  Source SourceRecord `json:"source,omitempty"`

}

```

Event定义的是具体的事件，Envelope定义的是事件的集合，也就是说在一次发送中可能会推送多个event。这两个struct也是我们会在下面的代码中直接用到的，



来看一下代码吧，

```

package main



import (

  "encoding/json"

  "fmt"

  "github.com/docker/distribution/notifications"

  "net/http"

)



// public

// map类型中的value为函数，使用key可以直接获取函数进行调用

var DIS_RULE = map[string]func(http.ResponseWriter, *http.Request, notifications.Event){

  "pull":  processPullEvent,

  "push":  processPushEvent,

  "other": processOtherEvent,

}



func processPushEvent(w http.ResponseWriter, r *http.Request, e notifications.Event) {

  fmt.Println("Push")

  fmt.Printf("%s n", e)

}



func processPullEvent(w http.ResponseWriter, r *http.Request, e notifications.Event) {

  fmt.Println("Pull")

  fmt.Printf("%s n", e)

}



func processOtherEvent(w http.ResponseWriter, r *http.Request, e notifications.Event) {

  fmt.Println("Other")

  fmt.Printf("%s n", e)

}



// public end



type Dispatcher struct {

  disRule map[string]func(http.ResponseWriter, *http.Request, notifications.Event)

}



type Server struct {

  dispatcher Dispatcher

}

// 处理分发

func (server Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {

  all_event := notifications.Envelope{}

  // json数据转化为本地的struct

  json_decoder := json.NewDecoder(r.Body)

  err := json_decoder.Decode(&amp;all_event)

  if err != nil {

    // panic()

  }



  // process events

  for _, event := range all_event.Events {

    switch event.Action {

    case "pull":

      server.dispatcher.disRule["pull"](w, r, event)

    case "push":

      server.dispatcher.disRule["push"](w, r, event)

    default:

      server.dispatcher.disRule["other"](w, r, event)



    }

  }

}



func newServer() Server {

  dis := Dispatcher{DIS_RULE}

  server := Server{dis}

  return server

}



func main() {

  var s Server

  s = newServer()

  // http server

  http.ListenAndServe(":1234", s)

}

```

上面的代码只是做了很简单的处理，建立http监听，收到数据后逐个处理event。dispatcher根据event的类型不同分发到不同的处理函数中。由于只是简单的例子，所以每个函数只是打印出了一些数据。如果真的需要用作监控，那么就需要将这些数据进行提取分类，然后进行持久化，已经有一些开源项目完成了持久化的工作，比如<a href="https://github.com/kwk/docker-registry-event-collector">docker-registry-event-collector</a>。



还有对于新人需要注意的是，在建立http监听的时候地址需要写成":1234"，这代表对于本地所有设备都启用1234端口进行监听。如果写成localhost:1234的话在其他机器上是无法Ping通这台机器的。



我们来运行下看看实际效果吧，

```
Pull

{a6b29455-b663-4843-a0e6-004ee5ae6484 2015-11-07 01:34:34.257422044 +0000 UTC pull {sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4} %!s(int64=32) ubuntu http://127.0.0.1:5000/v2/ubuntu/blobs/sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4} {ad8ccdd7-666e-42af-972e-315b5daa4ab4 172.17.42.1:50079 127.0.0.1:5000 GET docker/1.8.3 go/go1.4.2 git-commit/f4bf5c7 kernel/3.16.0-51-generic os/linux arch/amd64} {} {6c41a5e8cc0d:5000 6733ebcd-15b7-4f16-a072-a9b4892b9368}} 

Pull

{d8ad7b5a-e770-492b-913e-62e115f8a561 2015-11-07 01:34:34.258076116 +0000 UTC pull {sha256:916b974d99af866381ea9e3c929b4709058946bb44f3ad10dacfc6ea3b2a936b} %!s(int64=682) ubuntu http://127.0.0.1:5000/v2/ubuntu/blobs/sha256:916b974d99af866381ea9e3c929b4709058946bb44f3ad10dacfc6ea3b2a936b} {96eaaee5-fde8-475b-a21e-d870f459a3d4 172.17.42.1:50080 127.0.0.1:5000 GET docker/1.8.3 go/go1.4.2 git-commit/f4bf5c7 kernel/3.16.0-51-generic os/linux arch/amd64} {} {6c41a5e8cc0d:5000 6733ebcd-15b7-4f16-a072-a9b4892b9368}}

```

如果需要运行上面的代码，需要有一些go的基本常识，比如如何将github的包引用下来等，这里就不赘述了。



Thank you

