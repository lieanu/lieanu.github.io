---
layout: post
title: MIT 6.824 lab2 笔记

---

> 笔记很乱,作业很痛苦... 

Primary必须将Gets和Puts都发给Backup,在给client回复之前,应该先等Backup的回复.  

假如有2个服务器,S1是主,S2是备,Viewservice认为S1挂了(可能出错了),然后S2会顶上,从备份的身份提升到Primary的身份.而Client不认为S1挂了,依旧给S1发操作,S1把操作转给S2,但S2给回了个错,告诉它我已经不是备份了,那么S1就会给client返回一个错告诉它,我自己现在已经不是Pirmary了.原因是S2拒绝了我给的操作,那么肯定已经有了一个新的View了.这时Client会找Viewservice要正确的View(S2是Primary).  k/v server把数据保存在内存里,而不是磁盘里,所以当server挂了重启后,数据会丢.如果这时没有Backup,那么Primary挂了重启后,它不能再做Primary.数据全丢了.  这个实验的一些限制: 
* Viewservice不能挂,它没备份.
* Primary和Backup一次只处理一个操作,限制了它的performance.
* 一个重新恢复的Server要从Primary那里拷贝完整的一个数据库,这可能会相当的慢.
* server把数据存在内存里,如果它们同时挂了,那就真的挂了.
* 主到备不能通信时,不好办.
* 如果Primary不对当前的View做ACK,那么Viewservice不能改View,不能往前走.

1. primary挂了,就提升backup为primary.
2. backup挂了,或者被提升了,那就找个空闲的server做backup.
3. primary把所有的数据发给新的backup,再发送后续的操作给backup,来确保数据的一致性.


##Part A: View Service

1. ViewService只有一个,没有副本.
2. 一个view里的Primary,必须是上一个view的primary或backup.
3. 一个例外:当viewservice启动时,它必须接收任意一个服务做为第一个primary.
4. backup可以是其它的任意一个server,如果没有可用的server,那就是"".
5. 每个k/v server应该在每个Pinginterval周期内,发送一个Ping RPC.
6. Viewservice将当前的View做为回复.
7. Ping使得viewservice知道这个server还活着,并将现在的view通报给这个server,并且也告诉viewservice,这个server知道的最近的view.
8. 如果Viewservice没有在Deadpings Pingintervals周期内,收到某个服务器的Ping的话,那么可以认为这个服务器挂了.
9. 当服务器重启后,它发送一个或多个参数为0的Ping,来通知Viewservice它挂了(Ping(0)).
10. Viewservice在这种情况下发起一个新的View: a.没有收到primary和backup的Ping时.b.primary或backup挂了重启后.c.没有backup,但有一个空闲的server时.
11. Viewservice在Primary没有确认当前view的情况下,不能做viewchange.
12. 如果viewservice没有从primary收到当前view的确认,那么viewservice不应该改做viewchange,即使它认为primary和backup都挂了.

Hint:

1. 在viewservice结构里,为每个server添加最近的view,使用map.
2. 在viewservice结构里,记录当前的view.
3. 记录当前view是否被primary确认.
4. Viewserver做周期性的检查判定,比如primary挂了backup要顶上,这些在tick()函数里实现.
5. 多个Server发Ping,多余的可以在需要时,自愿成为backup.(PS.需要时,应该就是Backup挂了时吧).
6. viewservice需要一种能检查primary或backup挂了重启的方式.比如,primary可能挂了,然后又重启,但没有漏过发送Ping.
7. Coding前,先看测试.

Here is the code:

```go
func (vs *ViewServer) Ping(args *PingArgs, reply *PingReply) error {

	// Your code here.
	if vs.currentView.Primary == args.Me && vs.currentView.Viewnum == args.Viewnum {
		vs.ACK = true
	}

	vs.recTime[args.Me] = time.Now()

	if args.Viewnum == 0 {
		if vs.currentView.Primary == "" && vs.currentView.Backup == "" {
			vs.currentView.Primary = args.Me
			vs.currentView.Viewnum = 1
			vs.ACK = true
		} else if vs.currentView.Primary == args.Me {
			vs.recTime[args.Me] = time.Time{}
		} else if vs.currentView.Backup == args.Me {
			vs.recTime[args.Me] = time.Time{}
		}
	}
	vs.sync1 <- 1
	<-vs.sync2
	reply.View = vs.currentView

	return nil
}

//
// server Get() RPC handler.
//
func (vs *ViewServer) Get(args *GetArgs, reply *GetReply) error {

	// Your code here.
	reply.View = vs.currentView
	return nil
}

//
// tick() is called once per PingInterval; it should notice
// if servers have died or recovered, and change the view
// accordingly.
//
func (vs *ViewServer) tick() {

	// Your code here.
	<-vs.sync1
	for server, _ := range vs.recTime {
		if time.Now().Sub(vs.recTime[server]) > (DeadPings * PingInterval) {
			delete(vs.recTime, server)
			if vs.ACK {
				if server == vs.currentView.Primary {
					vs.currentView.Primary = ""
				} else if server == vs.currentView.Backup {
					vs.currentView.Backup = ""
				}
			}
		}
	}

	flag := false

	if vs.ACK {
		if vs.currentView.Primary == "" && vs.currentView.Backup != "" {
			vs.currentView.Primary = vs.currentView.Backup
			vs.currentView.Backup = ""
			flag = true
		}

		if vs.currentView.Backup == "" {
			bakserver := ""
			for s, _ := range vs.recTime {
				if s != vs.currentView.Primary {
					bakserver = s
					break
				}
			}
			if bakserver != "" {
				vs.currentView.Backup = bakserver
				flag = true
			}
		}

		if flag {
			vs.currentView.Viewnum += 1
			vs.ACK = false
		}
	}
	vs.sync2 <- 1
}
```

##Part B: Shared Key/Value Server

* Partition
* as long as no server

Get()返回最新的value.Put()存Value,PutHash().

Client和Server之间只能通过RPC接口进行通信.

任何时候只有一个Primary.假如某个View里S1是primary,做了个viewchange后,S2变成Primary了,但是S1还不知道新的View,它还认为自己是Primary.所以有一些client找S1,有一些Client找S2.

不是Primary的Server,就不要给Client任何答复,或者给个出错的答复.

Get(),Put(),PutHash()只有在操作完成后,才返回(这不废话吗).Put应该在更新了k/v database后返回.Get()应该在它取到了相应的Value以后返回.commit point.

server不应该在每次收到Put/Get后,跟Viewservice联系.只有定时Ping Viewservice即可.

如果Backup提升到Primary了,那么老的Primary再给Backup发消息,它就给拒绝掉就行了.

如何开始:

1. server.go开始,Ping Viewservice来找最新的View.在tick()函数里干这个事?(why).server知道View以后,那么他也就知道它的身份了.
2. 完成Put()和Get()函数.把所有的key/value存在map[string]string里.k/v 存在`map[string] string`内.`strconv.Itoa(int(h))`要转换Hash后的结果.
3. 改动Put(),让Primary可以把更新转给backup.
4. 当新的server成了backup了,那么要把primary里的数据都发给backup.
5. 改动client.go,client可以从一个挂掉的primary中切出来.如果当前的primary不回复,或者我不认为它是primary,client就会向viewservice咨询,然后再次尝试.


