
title: marathon---event-bus
date: 2017-09-07 14:55:44
tags: [youdaonote]
---

marathon有一个内部的event bus，处理所有的API请求和扩容event。通过订阅这个eventbus，我们可以实时了解发生的所有事件。event bus对于整合基于状态的实体很有用处，比如load balancer，或者compile statistics。

Evet可通过插件化的subscriber进行订阅。

event bus有两个API：
 - event stream。详情见/v2/events/
 - 回调endpoint，以json的方式把event发送到一个或多个endpoint中。这个方式已经过期了。
 
event stream相对好处
- 容易设置
- 传递较快。没有req/resp的处理
- event是有序的

#### 回调方式
```
./bin/start --master ... --event_subscriber http_callback --http_endpoints http://host1/foo,http://host2/bar
```


#### event类型
##### API请求
marathon接收到修改某个app的时候收到的API请求。
```
{
  "eventType": "api_post_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "clientIp": "0:0:0:0:0:0:0:1",
  "uri": "/v2/apps/my-app",
  "appDefinition": {
    "args": [],
    "backoffFactor": 1.15,
    "backoffSeconds": 1,
    "cmd": "sleep 30",
    "constraints": [],
    "container": null,
    "cpus": 0.2,
    "dependencies": [],
    "disk": 0.0,
    "env": {},
    "executor": "",
    "healthChecks": [],
    "id": "/my-app",
    "instances": 2,
    "mem": 32.0,
    "ports": [10001],
    "requirePorts": false,
    "storeUrls": [],
    "upgradeStrategy": {
        "minimumHealthCapacity": 1.0
    },
    "uris": [],
    "user": null,
    "version": "2014-09-09T05:57:50.866Z"
  }
}
```
##### 状态更新

task的状态变更时发送的event:
```
{
  "eventType": "status_update_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "slaveId": "20140909-054127-177048842-5050-1494-0",
  "taskId": "my-app_0-1396592784349",
  "taskStatus": "TASK_RUNNING",
  "appId": "/my-app",
  "host": "slave-1234.acme.org",
  "ports": [31372],
  "version": "2014-04-04T06:26:23.051Z"
}
```

所有的状态
 - TASK_STAGING
 - TASK_STARTING
 - TASK_RUNNING
 - TASK_FINISHED
 - TASK_FAILED
 - TASK_KILLING
 - TASK_KILLED
 - TASK_LOST
后面四种是终止状态。

##### framework message
```
{
  "eventType": "framework_message_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "slaveId": "20140909-054127-177048842-5050-1494-0",
  "executorId": "my-app.3f80d17a-37e6-11e4-b05e-56847afe9799",
  "message": "aGVsbG8gd29ybGQh"
}
```

##### Event订阅
订阅者消息有变化时触发。
```
{
  "eventType": "subscribe_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "clientIp": "1.2.3.4",
  "callbackUrl": "http://subscriber.acme.org/callbacks"
}
```
```
{
  "eventType": "unsubscribe_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "clientIp": "1.2.3.4",
  "callbackUrl": "http://subscriber.acme.org/callbacks"
}
```

##### Health Check
```
{
  "eventType": "add_health_check_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "appId": "/my-app",
  "healthCheck": {
    "protocol": "HTTP",
    "path": "/health",
    "portIndex": 0,
    "gracePeriodSeconds": 5,
    "intervalSeconds": 10,
    "timeoutSeconds": 10,
    "maxConsecutiveFailures": 3
  }
}
```

```
{
  "eventType": "remove_health_check_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "appId": "/my-app",
  "healthCheck": {
    "protocol": "HTTP",
    "path": "/health",
    "portIndex": 0,
    "gracePeriodSeconds": 5,
    "intervalSeconds": 10,
    "timeoutSeconds": 10,
    "maxConsecutiveFailures": 3
  }
}
```

```
{
  "eventType": "failed_health_check_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "appId": "/my-app",
  "taskId": "my-app_0-1396592784349",
  "healthCheck": {
    "protocol": "HTTP",
    "path": "/health",
    "portIndex": 0,
    "gracePeriodSeconds": 5,
    "intervalSeconds": 10,
    "timeoutSeconds": 10,
    "maxConsecutiveFailures": 3
  }
}
```

```
{
  "eventType": "health_status_changed_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "appId": "/my-app",
  "instanceId": "my-app.instance-c7c311a4-b669-11e6-a48f-0ea4f4b1778c",
  "version": "2014-04-04T06:26:23.051Z",
  "alive": true
}
```

```
{
  "appId": "/my-app",
  "taskId": "my-app_0-1396592784349",
  "version": "2016-03-16T13:05:00.590Z",
  "reason": "500 Internal Server Error",
  "host": "localhost",
  "slaveId": "4fb620fa-ba8d-4eb0-8ae3-f2912aaf015c-S0",
  "eventType": "unhealthy_task_kill_event",
  "timestamp": "2016-03-21T09:15:10.764Z"
}
```

##### deployment

```
{
  "eventType": "group_change_success",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "groupId": "/product-a/backend",
  "version": "2014-04-04T06:26:23.051Z"
}
```

```
{
  "eventType": "group_change_failed",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "groupId": "/product-a/backend",
  "version": "2014-04-04T06:26:23.051Z",
  "reason": ""
}
```


```
{
  "eventType": "deployment_success",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "id": "867ed450-f6a8-4d33-9b0e-e11c5513990b"
}
```

```
{
  "eventType": "deployment_failed",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "id": "867ed450-f6a8-4d33-9b0e-e11c5513990b"
}
```

```
{
  "eventType": "deployment_info",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "plan": {
    "id": "867ed450-f6a8-4d33-9b0e-e11c5513990b",
    "original": {
      "apps": [],
      "dependencies": [],
      "groups": [],
      "id": "/",
      "version": "2014-09-09T06:30:49.667Z"
    },
    "target": {
      "apps": [
        {
          "args": [],
          "backoffFactor": 1.15,
          "backoffSeconds": 1,
          "cmd": "sleep 30",
          "constraints": [],
          "container": null,
          "cpus": 0.2,
          "dependencies": [],
          "disk": 0.0,
          "env": {},
          "executor": "",
          "healthChecks": [],
          "id": "/my-app",
          "instances": 2,
          "mem": 32.0,
          "ports": [10001],
          "requirePorts": false,
          "storeUrls": [],
          "upgradeStrategy": {
              "minimumHealthCapacity": 1.0
          },
          "uris": [],
          "user": null,
          "version": "2014-09-09T05:57:50.866Z"
        }
      ],
      "dependencies": [],
      "groups": [],
      "id": "/",
      "version": "2014-09-09T05:57:50.866Z"
    },
    "steps": [
      {
        "action": "ScaleApplication",
        "app": "/my-app"
      }
    ],
    "version": "2014-03-01T23:24:14.846Z"
  },
  "currentStep": {
    "actions": [
      {
        "type": "ScaleApplication",
        "app": "/my-app"
      }
    ]
  }
}
```
