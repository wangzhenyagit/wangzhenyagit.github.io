---
layout: post
title: mangos（一）概述与消息处理机制
category: 开源研究
tags: mangos
---

## 概述
想看下开源的服务器框架，本以为挺复杂，但mangos代码写的很清楚。mangos不是一个魔兽私服模拟器，它是一个开源的自由软件项目，是用c++和C#编程语言，实现的一个支持大型多人在线角色扮演游戏服务器的程序框架。svn的路径：http://svn.code.sf.net/p/mangos/code/trunk 下载下来貌似有100多兆，我用的vs2005编译vc8工程release版本一次就编译过了。

主目录中文件夹有：
contrib 第三方的工具
dep 依赖的开源库ace sqlite等
src 项目代码
sql 数据库脚本

src目录下文件夹有：
bindings文件夹中包含脚本文件，应该是对脚本进行绑定的。
framework文件夹中包括一些游戏框架，其中包括网络框架，游戏系统框架，工具，平台等内容。
game文件夹中应该是游戏的文件，包括世界系统，战斗系统，游戏事件，游戏场景等的实现。
mangosd文件夹中是mangosd的主程序，包括程序的入口等。
realmd 文件夹中是游戏区域信息，包括RealmList等内容。
shared文件夹中 应该是公用的函数和库，database的内容包含在其中。

线程分布： 

1、主线程 main---- 主要功能：初始化world、创建子线程、回收资源 

2、WorldRunnable -------主线程 

3、CliRunnable -----调试线程 command line 

4、RARunnable -------Remote Administration 处理远程管理命令？

5、MaNGOSsoapRunnable---协议 

6、FreezeDetectorRunnable ---- 心跳检测

7、SqlDelayThread --- 数据线程

8、PatcherRunnable ---- 给客户端升级（发送补丁文件）

这里对于线程类的命名都是以Runnable开始，以继承的方式实现线程类，从而对线程的分布一目了然，并且对类有一定说明作用。

事件分发和处理：
WorldRunnable::run---World:update----World:UpdateSessions---WorldSession::Update(一个socket内所有事件)---各种各样的handler

## WorldRunnable类

```
/// Heartbeat for the World
void WorldRunnable::run()
{
    ///- Init new SQL thread for the world database
    WorldDatabase.ThreadStart();                                // let thread do safe mySQL requests (one connection call enough)
    sWorld.InitResultQueue();

    uint32 realCurrTime = 0;
    uint32 realPrevTime = getMSTime();

    uint32 prevSleepTime = 0;                               // used for balanced full tick time length near WORLD_SLEEP_CONST

    ///- While we have not World::m_stopEvent, update the world
    while (!World::m_stopEvent)
    {
        ++World::m_worldLoopCounter;
        realCurrTime = getMSTime();

        uint32 diff = getMSTimeDiff(realPrevTime,realCurrTime);

        sWorld.Update( diff );
        realPrevTime = realCurrTime;

        // diff (D0) include time of previous sleep (d0) + tick time (t0)
        // we want that next d1 + t1 == WORLD_SLEEP_CONST
        // we can't know next t1 and then can use (t0 + d1) == WORLD_SLEEP_CONST requirement
        // d1 = WORLD_SLEEP_CONST - t0 = WORLD_SLEEP_CONST - (D0 - d0) = WORLD_SLEEP_CONST + d0 - D0
        if (diff <= WORLD_SLEEP_CONST+prevSleepTime)
        {
            prevSleepTime = WORLD_SLEEP_CONST+prevSleepTime-diff;
            ZThread::Thread::sleep(prevSleepTime);
        }
        else
            prevSleepTime = 0;
    }

    ... // 清理资源
}
```

- 这是游戏世界的驱动线程，一开始很困惑“Heartbeat”难道是用来保持长连接用的？其实这里的Heartbeat应该理解为是整个世界的驱动的地方，像人的心脏，汽车的发动机之类。

- WorldDatabase.ThreadStart();这里名字很明确，“ThreadStart”，线程启动，如果内部带线程运行，名字中最好体现，不要叫做“Start”之类。同样结束线程名字为WorldDatabase.ThreadEnd(); 

- 在线程中，每次处理完都sleep了一段时间，有注释如下
// diff (D0) include time of previous sleep (d0) + tick time (t0)
// we want that next d1 + t1 == WORLD_SLEEP_CONST
// we can't know next t1 and then can use (t0 + d1) == WORLD_SLEEP_CONST requirement
// d1 = WORLD_SLEEP_CONST - t0 = WORLD_SLEEP_CONST - (D0 - d0) = WORLD_SLEEP_CONST + d0 - D0

这可能是游戏服务器中的特殊的部分，游戏服务器不需要“实时”处理用户的请求，只需要让用户觉得是“实时”的就够了。就像电视，不论是液晶的合适CRT的都有个刷新频率。而且如果使用“实时”处理的方式还会引入很多不必要的问题，例如，如果实时处理没有sleep，假设服务器能够处理，用户通过某种方法，在1秒内发送了1000次的“出拳”指令，如果不加以处理，那么一碰到这拳头其他人就挂了。这个固定的处理时间，也给整个游戏世界定了一个时间的最小片段，动作频率的最小片段，方便以后各种业务的处理。

sleep会不会浪费cpu呢？不会，因为只有在线程能够在一个WORLD_SLEEP_CONST处理完所有操作时候才会sleep，如果这个处理线程一直是满负荷的，那么这个线程也会一直工作。

计算方法看着比较复杂，其实不难。不考虑其他情况，假如线程每次都能处理完(diff<=WORLD_SLEEP_CONST+prevSleepTime)，
那么有DO = d0 + t0, d1 = WORLD_SLEEP_CONST + d0 - D0 = WORLD_SLEEP_CONST - t0;
那么假设WORLD_SLEEP_CONST为100ms，处理花费10ms(t0)，那么就sleep90ms(100 - 10)就可以了;然后，diff = 10 + 90 = 100ms,这种理想的情况下，diff一直是100ms，但是在线程很忙的时候，sWorld.Update时间大于100ms时候，也就是diff>WORLD_SLEEP_CONST+prevSleepTime <==> t0 + d0 >  WORLD_SLEEP_CONST+prevSleepTime(d0),即t0 > WORLD_SLEEP_CONST时候，diff就不等100了，这个时候prevSleepTime，diff等于sWorld.Update。

为什么需要计算diff这个参数呢？为了给sWorld.Update中的定时器提供时间参数，如果diff每次都是100ms，那么只传入一个m_worldLoopCounter即可，但是diff并不是一直都是100ms，所以需要把“过去多长时间”这个参数传入。相比定时器内部记录上次时间，每次轮询获取当前时间比较来说，这样使用一是简单些，最重要的是准确，如果不是这样使用，那么第二个定时器与第一个定时器由于调用顺序问题获取到的系统时间可能是不一样的，这样对于游戏玩家来说，时间就不同步了，同样是在一个世界里最小的时间片段内，为什么别人的时间就比我的快呢？

- 以后主要逻辑到sWorld.Update( diff ) 更新整个世界模型。

## World类

```
/// The World
class World
{
    public:
		...
        //player Queue
        typedef std::list<WorldSession*> Queue;
        void AddQueuedPlayer(WorldSession*);
        void RemoveQueuedPlayer(WorldSession*);
     
        void Update(time_t diff);

        void UpdateSessions( time_t diff );
      
        void ProcessCliCommands();
        void QueueCliCommand(CliCommandHolder* command) { cliCmdQueue.add(command); }

        void UpdateResultQueue();
        void InitResultQueue();
		...
    protected:
        void _UpdateGameTime();
        void ScriptsProcess();
        // callback for UpdateRealmCharacters
        void _UpdateRealmCharCount(QueryResult *resultCharCount, uint32 accountId);

        void InitDailyQuestResetTime();
        void ResetDailyQuests();
    private:
		...
        typedef HM_NAMESPACE::hash_map<uint32, WorldSession*> SessionMap;
        SessionMap m_sessions;
        std::set<WorldSession*> m_kicked_sessions;
        uint32 m_maxActiveSessionCount;
        uint32 m_maxQueuedSessionCount;

        std::multimap<time_t, ScriptAction> m_scriptSchedule;

        uint32 m_ShutdownTimer;
        uint32 m_ShutdownMask;
		...
        // CLI command holder to be thread safe
        ZThread::LockedQueue<CliCommandHolder*, ZThread::FastMutex> cliCmdQueue;
        SqlResultQueue *m_resultQueue;
        //Player Queue
        Queue m_QueuedPlayer;
        
        //sessions that are added async
        void AddSession_(WorldSession* s);
        ZThread::LockedQueue<WorldSession*, ZThread::FastMutex> addSessQueue;
		// 这里，用户添加不是直接添加到m_sessions列队里面，异步添加，先添加到一个临时队列，等待定时器到时候从队列取出放入m_sessions列队
};
```

```
/// Update the World !
void World::Update(time_t diff)
{
    ///- Update the different timers
    for(int i = 0; i < WUPDATE_COUNT; i++)
        if(m_timers[i].GetCurrent()>=0)
            m_timers[i].Update(diff);
    else m_timers[i].SetCurrent(0);

    ///- Update the game time and check for shutdown time
    _UpdateGameTime();

    /// Handle daily quests reset time
    if(m_gameTime > m_NextDailyQuestReset)
    {
        ResetDailyQuests();
        m_NextDailyQuestReset += DAY;
    }

    /// <ul><li> Handle auctions when the timer has passed
    if (m_timers[WUPDATE_AUCTIONS].Passed())
    {
		...
    }

    /// <li> Handle session updates when the timer has passed
    if (m_timers[WUPDATE_SESSIONS].Passed())
    {
        m_timers[WUPDATE_SESSIONS].Reset();

        UpdateSessions(diff);
    }

    /// <li> Handle weather updates when the timer has passed
    if (m_timers[WUPDATE_WEATHERS].Passed())
    {
       ...
    }
    /// <li> Update uptime table
    if (m_timers[WUPDATE_UPTIME].Passed())
    {
       ...
    }

    /// <li> Handle all other objects
    if (m_timers[WUPDATE_OBJECTS].Passed())
    {
       ...
    }

    // execute callbacks from sql queries that were queued recently
    UpdateResultQueue();

    ///- Erase corpses once every 20 minutes
    if (m_timers[WUPDATE_CORPSES].Passed())
    {
        m_timers[WUPDATE_CORPSES].Reset();

        CorpsesErase();
    }

    ///- Process Game events when necessary
    if (m_timers[WUPDATE_EVENTS].Passed())
    {
        m_timers[WUPDATE_EVENTS].Reset();                   // to give time for Update() to be processed
        uint32 nextGameEvent = gameeventmgr.Update();
        m_timers[WUPDATE_EVENTS].SetInterval(nextGameEvent);
        m_timers[WUPDATE_EVENTS].Reset();
    }

    /// </ul>
    ///- Move all creatures with "delayed move" and remove and delete all objects with "delayed remove"
    MapManager::Instance().DoDelayedMovesAndRemoves();

    // update the instance reset times
    sInstanceSaveManager.Update();

    // And last, but not least handle the issued cli commands
    ProcessCliCommands();
}
void World::UpdateSessions( time_t diff )
{
    while(!addSessQueue.empty())
    {
      WorldSession* sess = addSessQueue.next ();
      AddSession_ (sess);
    }
        
    ///- Delete kicked sessions at add new session
    for (std::set<WorldSession*>::iterator itr = m_kicked_sessions.begin(); itr != m_kicked_sessions.end(); ++itr)
        delete *itr;
    m_kicked_sessions.clear();

    ///- Then send an update signal to remaining ones
    for (SessionMap::iterator itr = m_sessions.begin(), next; itr != m_sessions.end(); itr = next)
    {
        next = itr;
        ++next;

        if(!itr->second)
            continue;

        ///- and remove not active sessions from the list
        if(!itr->second->Update(diff))                      // As interval = 0
        {
            delete itr->second;
            m_sessions.erase(itr);
        }
    }
}
```

- Session的创建是在worldSocket中，而Session的释放是在word中，在World::UpdateSessions( time_t diff )时候如果检测到Session无效则释放。非常规但是有利于管理。

- 整个程序的有个基本原则，逻辑单线程，尽量少用锁。锁还是比不可少的，因为不能所有的处理都在tcp的reactor读回调线程中，根据生产者消费者模型，从缓冲区取得数据处理的时候是需要加锁的。其实锁多少也不是关键问题，使用锁首先不能让锁竞争时候等待时间过长，即持有锁的时间不能长，否则直接影响系统的吞吐量。其次锁竞争不要多。要做到上述条件即要求锁的粒度要小。其次锁多，不一定锁竞争就多。就像线程多，不一定线程的切换开销多一样，可能大多数线程是sleep的不需要切换。持有锁的时间足够短，即使访问锁频率很快也可能没有竞争问题，线程占用锁的时候这个锁是没有被持有的，如果持有锁时间短，那么在其他线程申请占用锁的时候锁已经被释放掉了。

由于tcp读线程与word的heartbeat线程不是一个线程，传递消息肯定会涉及到锁了。这两个线程使用锁的地方有：
1）ZThread::LockedQueue<WorldSession*, ZThread::FastMutex> addSessQueue 这个queue存在也是为了减少锁的范围，减少锁竞争，当有Session增加的时候，不是直接放到world的Session队列里面而是放入一个临时的addSessQueue队列，然后下次轮询的时候一次性把缓存的消息放入Session队列中。
2）另一个地方在WorldSession中 ZThread::LockedQueue<WorldPacket*,ZThread::FastMutex> _recvQueue;这个在WorldSocket::ProcessIncoming中会调用m_Session->QueuePacket (new_pct);方法放入这个_recvQueue队列，然后再WorldSession::Update时候从队列中依次取出处理。

为了提高效率，可以在WorldSession::Update中拷贝出所有的消息，然后再while中处理，这种做法减少了锁竞争的可能性（其实效果没啥，在100ms内用户能做多少次操作呢，而且是读一个释放一次锁），网上文章中也有类似的方法，搞两个队列，交换处理队列，代替拷贝原理差不多。目的都是为了减少锁竞争，但是这样做有个问题，虽然减少了锁竞争，但处理过程中收到的用户操作消息就要延迟处理。如果拷贝队列的方式，假设用户发出操作a0的时候刚刚拷贝了整个队列，执行50ms,sleep50ms,然后再处理a0操作，需要50ms执行完成，那么a0的执行时间对用户来说可能是150ms，如果100ms对用户感觉是个卡的临界那么就会感觉到卡顿。相反，如果在updata过程中可以加入消息，那么可以在本次轮询执行完成。最坏情况，在刚开始sleep的时候a0到达，那么也只需要50ms的sleep+50ms的执行共100ms了。就不会感到卡顿。

## WorldSession类

```
/*
* A FastMutex is a small fast implementation of a non-recursive, mutually exclusive
* Lockable object. This implementation is a bit faster than the other Mutex classes
* as it involved the least overhead. However, this slight increase in speed is 
* gained by sacrificing the robustness provided by the other classes. 
*
* A FastMutex has the useful property of not being interruptable; that is to say  
* that acquire() and tryAcquire() will not throw Interrupted_Exceptions.
*/

/// Player session in the World
class MANGOS_DLL_SPEC WorldSession
{
    public:
        void QueuePacket(WorldPacket* new_packet);
        bool Update(uint32 diff);

    public:                                                 // opcodes handlers

        void Handle_NULL(WorldPacket& recvPacket);          // not used
        void Handle_EarlyProccess( WorldPacket& recvPacket);// just mark packets processed in WorldSocket::OnRead
        ...// 为阅读方便删除很多handle方法
        void HandleGuildBankSetTabText(WorldPacket& recv_data);
		
	private:
		...
		Player *_player;
        WorldSocket *m_Socket;
        ZThread::LockedQueue<WorldPacket*,ZThread::FastMutex> _recvQueue; // 这里使用非递归锁，可以加快速度
};

/// Update the WorldSession (triggered by World update)
bool WorldSession::Update(uint32 /*diff*/)
{
  if (m_Socket)
    if (m_Socket->IsClosed ())
      { 
        m_Socket->RemoveReference (); // 操作引用计数来表示对象释放，没有使用智能指针
        m_Socket = NULL;
      }
  
    WorldPacket *packet;

    ///- Retrieve packets from the receive queue and call the appropriate handlers
    /// \todo Is there a way to consolidate the OpcondeHandlerTable and the g_worldOpcodeNames to only maintain 1 list?
    /// answer : there is a way, but this is better, because it would use redundant RAM
    while (!_recvQueue.empty())
    {
        packet = _recvQueue.next();

        /*#if 1
        sLog.outError( "MOEP: %s (0x%.4X)",
                        LookupOpcodeName(packet->GetOpcode()),
                        packet->GetOpcode());
        #endif*/

        if(packet->GetOpcode() >= NUM_MSG_TYPES)
        {
            sLog.outError( "SESSION: received non-existed opcode %s (0x%.4X)",
                LookupOpcodeName(packet->GetOpcode()),
                packet->GetOpcode());
        }
        else
        {
            OpcodeHandler& opHandle = opcodeTable[packet->GetOpcode()]; // 非主流，提高性能
            switch (opHandle.status)
            {
                case STATUS_LOGGEDIN:
                    if(!_player)
                    {
                        // skip STATUS_LOGGEDIN opcode unexpected errors if player logout sometime ago - this can be network lag delayed packets
                        if(!m_playerRecentlyLogout)
                            logUnexpectedOpcode(packet, "the player has not logged in yet");
                    }
                    else if(_player->IsInWorld())
                        (this->*opHandle.handler)(*packet);
                    // lag can cause STATUS_LOGGEDIN opcodes to arrive after the player started a transfer
                    break;
                case STATUS_TRANSFER_PENDING:
                    if(!_player)
                        logUnexpectedOpcode(packet, "the player has not logged in yet");
                    else if(_player->IsInWorld())
                        logUnexpectedOpcode(packet, "the player is still in world");
                    else
                        (this->*opHandle.handler)(*packet);
                    break;
                case STATUS_AUTHED:
                    m_playerRecentlyLogout = false;
                    (this->*opHandle.handler)(*packet);
                    break;
                case STATUS_NEVER:
                    sLog.outError( "SESSION: received not allowed opcode %s (0x%.4X)",
                        LookupOpcodeName(packet->GetOpcode()),
                        packet->GetOpcode());
                    break;
            }
        }

        delete packet;
    }

    ///- If necessary, log the player out
    time_t currTime = time(NULL);
    if (!m_Socket || (ShouldLogOut(currTime) && !m_playerLoading))
        LogoutPlayer(true);

    if (!m_Socket)
        return false;                                       //Will remove this session from the world session map

    return true;
}
```

- 这个WorldSession::Update就是系统的hotpath，限制性能的地方。没一句都会影响到系统的性能。这里为了提高性能，从写法上，有点走偏锋的意思，例如：
OpcodeHandler& opHandle = opcodeTable[packet->GetOpcode()];
1）opcodeTable结构体数组，而不是map，数组存取速度肯定比map快。
2）结构体定义如下
struct OpcodeHandler
{
    char const* name;
    SessionStatus status;
    void (WorldSession::*handler)(WorldPacket& recvPacket);
};

一个名字说明，一个操作时候状态，一个处理函数指针。真正非主流的做法就是直接拿到一个类中的成员函数指针，然后通过此函数指针调用方法。如果使用boost的人看到这一般会bind下，但是这样做的话都会有对象声明周期管理的开销，尤其是boost的bind方法（从bind到释放前后调用六七次构造析构函数）。上面的写法可以快速找到对应的处理方法直接调用，通常来说这么多处理方法经常写程序的人可能会封装成各种类对象，然后对不同的opcode创建对象来处理，这样与上面一样有对象生命周期管理的开销。
这样做可以把复杂的容易变动的地方封装但不影响性能（如果写个大型的switch case也可以效率应该差不多，可读性差，而且会经常修改）。

这种用法要特别注意，如果对象没有实例化（上面已经实例化，在对象内调用自己的public方法），同样也可以通过类的成员函数指针调用成员函数，但是需要特别注意这个调用不能操作任何数据对象，对象没有实例化，没有内存。也非静态函数不能调用静态数据成员。

_recvQueue中的消息是m_Socket封包放入的。


## WorldSocket类

```
/**
 * WorldSocket.
 * 
 * This class is responsible for the comunication with 
 * remote clients.
 * Most methods return -1 on failure. 
 * The class uses refferece counting.
 *
 * For output the class uses one buffer (64K usually) and 
 * a queue where it stores packet if there is no place on 
 * the queue. The reason this is done, is because the server 
 * does realy a lot of small-size writes to it, and it doesn't 
 * scale well to allocate memory for every. When something is 
 * writen to the output buffer the socket is not immideately 
 * activated for output (again for the same reason), there 
 * is 10ms celling (thats why there is Update() method). 
 * This concept is simmilar to TCP_CORK, but TCP_CORK 
 * usses 200ms celling. As result overhead generated by 
 * sending packets from "producer" threads is minimal, 
 * and doing a lot of writes with small size is tollerated.
 * 
 * The calls to Upate () method are managed by WorldSocketMgr
 * and ReactorRunnable.
 * 
 * For input ,the class uses one 1024 bytes buffer on stack 
 * to which it does recv() calls. And then recieved data is 
 * distributed where its needed. 1024 matches pritey well the 
 * traffic generated by client for now.
 *  
 * The input/output do speculative reads/writes (AKA it tryes 
 * to read all data avaible in the kernel buffer or tryes to 
 * write everything avaible in userspace buffer), 
 * which is ok for using with Level and Edge Trigered IO 
 * notification.
 * 
 */
class WorldSocket : protected WorldHandler
{
public:
  /// Add refference to this object.
  long AddReference (void);

  /// Remove refference to this object.
  long RemoveReference (void);
  
  int ProcessIncoming (WorldPacket* new_pct);

};

//关键函数
int WorldSocket::ProcessIncoming (WorldPacket* new_pct)
{
    ACE_ASSERT (new_pct);
  
    // manage memory ;)
    ACE_Auto_Ptr<WorldPacket> aptr (new_pct);

    const ACE_UINT16 opcode = new_pct->GetOpcode ();

    if (this->closing_)
        return -1;

    // dump recieved packet
    if (sWorldLog.LogWorld ())
    {
        sWorldLog.Log ("CLIENT:\nSOCKET: %u\nLENGTH: %u\nOPCODE: %s (0x%.4X)\nDATA:\n",
                     (uint32) get_handle (),
                     new_pct->size (),
                     LookupOpcodeName (new_pct->GetOpcode ()),
                     new_pct->GetOpcode ());

        uint32 p = 0;
        while (p < new_pct->size ())
        {
            for (uint32 j = 0; j < 16 && p < new_pct->size (); j++)
                sWorldLog.Log ("%.2X ", (*new_pct)[p++]);
            sWorldLog.Log ("\n");
        }
        sWorldLog.Log ("\n\n");
    }

    // like one switch ;)
    if (opcode == CMSG_PING)
    {
        return HandlePing (*new_pct);
    }
    else if (opcode == CMSG_AUTH_SESSION)
    {
        if (m_Session)
        {
            sLog.outError ("WorldSocket::ProcessIncoming: Player send CMSG_AUTH_SESSION again");
            return -1;
        }

        return HandleAuthSession (*new_pct);
    }
    else if (opcode == CMSG_KEEP_ALIVE)
    {
        DEBUG_LOG ("CMSG_KEEP_ALIVE ,size: %d", new_pct->size ());

        return 0;
    }
    else
    {
        ACE_GUARD_RETURN (LockType, Guard, m_SessionLock, -1);

        if (m_Session != NULL)
        {
            // OK ,give the packet to WorldSession
            aptr.release ();
            // WARNINIG here we call it with locks held.
            // Its possible to cause deadlock if QueuePacket calls back
            m_Session->QueuePacket (new_pct);
			// 这里，向
            return 0;
        }
        else
        {
            sLog.outError ("WorldSocket::ProcessIncoming: Client not authed opcode = ", opcode);
            return -1;
        }
    }

    ACE_NOTREACHED (return 0);
}
```

这个类在封包后会调用ProcessIncoming这个方法，此时上线文还是在ACE_Reactor中唯一的一个读事件处理线程中，如果此处一阻塞所有的tcp的读就阻塞了。但是这里也处理了一个业务HandleAuthSession，这个处理登录的方法里面还有查询数据库等操作，有点费时间。但是如果登录数目不大的话，还可以理解。也可以理解这个操作是“十分重要”的，没有用户登录后面操作都没有任何意义。所以数目不大又重要的消息，直接在tcp线程中处理了。

## 总结
1. 为了提高系统性能，代码中并不够“面向对象”，很多地方switch case，甚至利用了一个不常用的方法例如通过函数指针调用类类方法。代码中很少有new delete操作。没有share_ptr智能指针，有部分使用auto_ptr。
2. 对于游戏服务器，单机能支持5000连接已经不错，关键看在一个处理周期最长时间100ms是否能够处理全部的请求。
3. IO操作单开线程，其他逻辑上处理单线程，减少线程切换、锁竞争开销，可以充分的利用cpu。设计到IO的地方不多，网络接收，这块ACE的Reactor都做好了，其次读写数据库需要一个线程，剩余大部分的操作都是逻辑操作。
4. 在处理数据库操作的时候，有复杂的操作是放入单独的线程中处理的，简单的就在轮询线程中处理掉了。对于放入单独线程处理的地方，在更新完内存并放入数据库处理线程后就直接返回成功了。目前未看到处理失败的处理机制，看来mysql还是很靠谱的了。

