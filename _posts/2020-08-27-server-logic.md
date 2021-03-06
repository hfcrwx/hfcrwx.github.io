---
title: "服务器逻辑"
published: true
---

# 服务器启动 #

## 1. Center - CE ##

    listen
        0.0.0.0:2001    inside
        0.0.0.0:9001    outside GM
    connect
        mysql   tt_center
        redis


    Work::Init                              测试MySQL连接

    UserManager::_loadUserInfoFromDB        MySQL : select COUNT(*) FROM user
                                            MySQL : select cPrenId, cMasterId from master_relation
                                            MySQL : select id,openId,nike,sex,token,channel,phone,provice,city,country,headImageUrl,unionId,numsCard1,numsCard2,numsCard3,lastLoginTime,regTime,new,gm,totalCardNum,totalPlayNum,coins,credits,ticket,winnerNum,moneyNum,zsCoinsConsume,diamond,isMove,score,totalScore,masterCode, playCount_0, winCount_0, maxMultiple_0, maxCardTypeDetail_0, playCount_1, winCount_1, maxMultiple_1, maxCardTypeDetail_1, playCount_2, winCount_2, maxMultiple_2, maxCardTypeDetail_2, playCount_3, winCount_3, maxMultiple_3, maxCardTypeDetail_3, userExp, masterScore, masterScoreUsed, cardRecordTime, limitCountFlag, lgBiggestScore,lgGetMoney,lgGetCoins,lgTotalNum FROM user ORDER BY id DESC LIMIT 0,32

    Work::m_RedisClient                     redis长连接 1
    Work::m_dbServer                        mysql长连接 1      onlinelog
    gDbServerManager.Start();               mysql长连接 16
    gDbNewUserManager.Start();              mysql长连接 4

**per 30s**

    //不是每次收到 CID_LM2CE_GATE_INFO 后直接调用，因为Center和LogicM的关系是一对多，防止频繁写入数据库
    //一次性写入多个LM上的用户在线信息，也便于对比
    Work::SaveCurrentOnline                 MySQL : <insert into onlinelog(dateTime, serverID, serverName, onlineCount) values('1573707152','15001','tt_sichuan','0')>

## 2. LogicDB - LDB ##

    listen
        0.0.0.0:7001    inside
    connect
        mysql   tt_logic
        redis


    Work::m_dbsession                       mysql长连接 1      seven_day_log
    CMailSystem::m_redisClient              redis长连接 1
    CMailSystem::m_dbsession                mysql长连接 1      mail

    CMailSystem::ClearRedisAtInit           MySQL : select cMailId from mail where cCreateTime < '2019-11-08'
    CMailSystem::DeleteOvertimeMail         MySQL : delete from mail where cCreateTime < '2019-11-08'
    CMailSystem::InitMultiReceMail          MySQL : select cMailId, cMailType, cSenderId, cSenderName, cMailTitle, cMailText, cCreateTime from mail where cReceiverId = 1
    Work::_loadSevenDayInfo                 MySQL : select userId, day, lastTime FROM seven_day_log

    gDbServerManager.Start();               mysql长连接 2
    gUserMessageMsg.Start();                mysql长连接 16
                                            redis长连接 16

## 3. LogicManager - LM ##

    listen
        0.0.0.0:10001   inside
    connect
        CE  2001
        LDB 7001
        redis


    //加载游戏配置
    Config::Init                            Redis : GET Config:Common:CoinToExpRate
                                            Redis : GET Config:Common:TicketToExpRate
                                            Redis : GET Config:LM:MaxDailyReportTimes
                                            Redis : HGET Config:LM:GameCfg FreePokerCardRecord
                                            Redis : HMGET Config:LM:GameCfg LaixiaMoveStartTime LaixiaMoveEndTime
                                            Redis : HGET Config:LM:GameCfg LaixiaMoveReward
                                            Redis : GET Config:Common:FriendStartCoins
                                            Redis : GET Config:Common:FriendEnterLimit
                                            Redis : GET ttmj:config:lm
                                            Redis : SMEMBERS ttmj:config:UIInfo:type
                                            Redis : HGETALL ttmj:config:UIInfo:gameLevel
                                            Redis : HGETALL ttmj:config:UIInfo:currencyType
                                            Redis : HGETALL ttmj:config:UIInfo:gameType
                                            Redis : HGETALL ttmj:config:UIInfo:gameMsg

    //任务
    CTaskManager::m_redisClient             redis长连接 1
    CTaskManager::Init                      Redis : HGET Config:LM:GameCfg SeniorTaskLevelLimit

    //师徒
    CMasterSystem::m_redisClient            redis长连接 1
    CMasterSystem::SetConfig                Redis : GET Config:LM:MasterSys

    //活动
    ActiveManager::m_redisClient            redis长连接 1
    CRuntimeInfoMsg::Init                   Redis : GET sys_horse_info

    //提现
    ExchangeMoneyManager::m_redisClient     redis长连接 1

    //排行榜
    RankManager::m_redisClient              redis长连接 1

    //连连看
    LinkGameManager::m_redisClient          redis长连接 1
    LinkGameManager::updateConfig           Redis : GET LinkGameConfig

    IPRegionManager::m_redis                redis长连接 1

**<1> LM connect CE : LSocketPtr m_centerClient**

    LM -> CE : CID_LM2CE_MASTER_RELATION
    CE :                                    MySQL : select cPrenId, cMasterId from master_relation // 其实CE启动时已经加载到程序内存里了
    CE -> LM : CID_CE2LM_MASTER_RELATION //加载游戏数据
    LM : CMasterSystem::InitData            Redis : SMEMBERS MasterSys:WakeUp
                                            Redis : HGET MasterSys:PrenInfo:10434890 TotalScoreOffer
                                            Redis : HGET MasterSys:PrenInfo:10434890 TotalRedPacketOffer
                                            Redis : HGET MasterSys:PrenInfo:10434890 OfflineTime
                                            Redis : HGET MasterSys:PrenInfo:10434890 PlayForRedPacket
                                            Redis : HGET MasterSys:PrenInfo:10434890 ActivityOffer
                                            Redis : HGET MasterSys:PrenInfo:10434890 CommisionOffer
                                            Redis : HGET MasterSys:PrenInfo:10888976 TotalScoreOffer
                                            Redis : HGET MasterSys:PrenInfo:10888976 TotalRedPacketOffer
                                            Redis : HGET MasterSys:PrenInfo:10888976 OfflineTime
                                            Redis : HGET MasterSys:PrenInfo:10888976 PlayForRedPacket
                                            Redis : HGET MasterSys:PrenInfo:10888976 ActivityOffer
                                            Redis : HGET MasterSys:PrenInfo:10888976 CommisionOffer
                                            Redis : HGETALL MasterSys:Activity:10198120
                                            Redis : HGETALL MasterSys:Activity:11482784
                                            Redis : SMEMBERS MasterSys:BigRedPacket:317
                                            Redis : SMEMBERS MasterSys:BigPacketGiven
                                            Redis : HGETALL MasterSys:RedPacketEarn:317
                                            Redis : HGETALL MasterSys:DailyPlay:317
                                            Redis : SMEMBERS MasterSys:DailyRedPacketInfo:317
                                            Redis : SMEMBERS MasterSys:BindRewardPacket:317
                                            Redis : SMEMBERS MasterSys:Activity:DailyGiven:317
                                            Redis : HGETALL MasterSys:LoginBind

    LM -> CE : CID_LM2CE_LOGIN
    CE : 保存 std::map<Lint, LSocketPtr> m_mapLogicManagerSp
    CE -> LM :
            //活动配置信息
            // CID_CE2LM_SET_GAME_FREE
            // CID_CE2LM_SET_PXACTIVE
            // CID_CE2LM_SET_OUGCACTIVE
            // CID_CE2LM_SET_EXCHACTIVE
            // CID_CE2LM_SET_MONEY_MATCH

            //UserInfos
            CID_CE2LM_USER_ID_INFO //分多次发送

            //排行榜
                                            MySQL : select id,coins FROM user ORDER BY coins DESC LIMIT 1000
            CID_CE2LM_TOP_USER_INFO //RANK_DETAIL_COINS 金币

                                            MySQL : select id,totalScore FROM user ORDER BY totalScore DESC LIMIT 1000
            CID_CE2LM_TOP_USER_INFO //RANK_DETAIL_SCORE 积分

                                            MySQL : select id,lgBiggestScore FROM user ORDER BY lgBiggestScore DESC LIMIT 1000
            CID_CE2LM_TOP_USER_INFO //RANK_DETAIL_LINK_GAME_SCORE 连连看积分

**<2> LM connect LDB : LSocketPtr m_dbClient**

    LM -> LDB : CID_LM2LDB_LOGIN
    LDB : 保存 LSocketPtr m_logicManager
                                            MySQL : select max(cMailId) from mail
    LDB -> LM : CID_LDB2LM_INIT_MAIL_ID

**per 30s**

    LM -> CE : CID_LM2CE_GATE_INFO
    CE :
        //这里没有直接 Work::SaveCurrentOnline，因为Center和LogicM的关系是一对多，防止频繁写入数据库
        保存 std::map<Lint, std::map<Lint, GateInfo> > m_mapGateInfo

**per 300s** //更新游戏配置

    Config::Init                            Redis : GET Config:Common:CoinToExpRate
                                            Redis : GET Config:Common:TicketToExpRate
                                            Redis : GET Config:LM:MaxDailyReportTimes
                                            Redis : HGET Config:LM:GameCfg FreePokerCardRecord
                                            Redis : HMGET Config:LM:GameCfg LaixiaMoveStartTime LaixiaMoveEndTime
                                            Redis : HGET Config:LM:GameCfg LaixiaMoveReward
                                            Redis : GET Config:Common:FriendStartCoins
                                            Redis : GET Config:Common:FriendEnterLimit
                                            Redis : GET ttmj:config:lm
                                            Redis : SMEMBERS ttmj:config:UIInfo:type
                                            Redis : HGETALL ttmj:config:UIInfo:gameLevel
                                            Redis : HGETALL ttmj:config:UIInfo:currencyType
                                            Redis : HGETALL ttmj:config:UIInfo:gameType
                                            Redis : HGETALL ttmj:config:UIInfo:gameMsg
    CMasterSystem::SetConfig                Redis : GET Config:LM:MasterSys

**per 600s**

    LM -> CE : CID_LM2CE_NOTICE_RANK_REFRESH
    CE -> LM :
            //排行榜
                                            MySQL : select id,coins FROM user ORDER BY coins DESC LIMIT 1000
            CID_CE2LM_TOP_USER_INFO //RANK_DETAIL_COINS 金币

                                            MySQL : select id,totalScore FROM user ORDER BY totalScore DESC LIMIT 1000
            CID_CE2LM_TOP_USER_INFO //RANK_DETAIL_SCORE 积分

                                            MySQL : select id,lgBiggestScore FROM user ORDER BY lgBiggestScore DESC LIMIT 1000
            CID_CE2LM_TOP_USER_INFO //RANK_DETAIL_LINK_GAME_SCORE 连连看积分

## 4. Coin - CO ##
    listen
        127.0.0.1:11001 inside
    connect
        LM 10001
        redis


    Config::Init                            Redis : GET Config:Common:CoinToExpRate
                                            Redis : GET Config:Common:TicketToExpRate
                                            Redis : HGETALL Config:Common:FlashCost
                                            Redis : GET Config:Common:FriendStartCoins
                                            Redis : GET Config:Common:FriendEnterLimit

    DeskManager::m_redisClient              redis长连接 1

**<1> CO connect LM : LSocketPtr m_logicManager**

    CO -> LM : CID_CO2LM_LOGIN
    LM : 保存 CoinsInfo m_coinsServer
    LM -> CO : CID_LM2CO_LOGIN
    // LM -> CO : CID_LM2CO_UPDATE_MONEY_MATCH_CONFIG

    // 0+ 如果最后加唯一的CO
    LM -> all G : CID_LM2L_LM2G_COINS_SERVER_INFO
    G connect CO : CoinsInfo m_coinsServer
    G -> CO : CID_G2CO_LOGIN
    CO : 保存 std::map<Lint, GateInfo> m_gateInfo

    // 0+ 如果最后加唯一的CO
    LM -> all L : CID_LM2L_LM2G_COINS_SERVER_INFO
    L connect CO : CoinsInfo m_coinsServer
    L -> CO : CID_L2CO_LOGIN
    CO : 保存 std::map<Lshort, std::map<Lint, LOGIC_SERVER_INFO> > m_logicServerInfo

**per 300s**

    Config::Init                            Redis : GET Config:Common:CoinToExpRate
                                            Redis : GET Config:Common:TicketToExpRate
                                            Redis : HGETALL Config:Common:FlashCost
                                            Redis : GET Config:Common:FriendStartCoins
                                            Redis : GET Config:Common:FriendEnterLimit

## 5. Logic - L ##
    listen
        127.0.0.1:6003  inside
    connect
        LM  10001
        LDB 7001
        redis


    Config::Init                            Redis : HGETALL Config:Common:FlashCost
    Work::redisClient_                      redis长连接 1

**<1> L connect LM : LSocketPtr m_logicManager**

    L -> LM : CID_L2LM_LOGIN
    LM : 保存 std::map<Lshort, std::map<Lint, LOGIC_SERVER_INFO> > m_logicServerInfo
    // LM -> L : CID_LM2L_PAIXING_ACTIVE
    // LM -> L : CID_LM2L_RLOG_INFO
    LM -> L : CID_LM2L_LM2G_COINS_SERVER_INFO

**<2> L connect LDB : LSocketPtr m_dbClient**

    L -> LDB : CID_L2LDB_LOGIN
    LDB : 没标记socket

**<3> L connect CO : CoinsInfo m_coinsServer**

    L -> CO : CID_L2CO_LOGIN
    CO : 保存 std::map<Lshort, std::map<Lint, LOGIC_SERVER_INFO> > m_logicServerInfo

    // 0+ 如果最后加一个或更多个L
    LM -> all G : CID_LM2G_SYNC_LOGIC
    G : connect L
    G -> L : CID_G2L_LOGIN
    L : 保存 std::map<Lint, GateInfo> m_gateInfo

## 6. Gate - G ##
    listen
        192.168.1.100:8001  outside
    connect
        LM  10001


**<1> G connect LM : LSocketPtr m_logicManager**

    G -> LM : CID_G2LM_LOGIN
    LM : 保存 std::map<Lint, GateInfo> m_gateInfo
    LM -> G : CID_LM2G_SYNC_LOGIC
    LM -> G : CID_LM2L_LM2G_COINS_SERVER_INFO
    LM -> CE : CID_LM2CE_GATE_INFO
    CE : 保存 std::map<Lint, std::map<Lint, GateInfo> > m_mapGateInfo

**<2> G connect all L : std::map<Lint, LogicInfo> m_logicInfo**

    G -> L : CID_G2L_LOGIN
    L : 保存 std::map<Lint, GateInfo> m_gateInfo

**<3> G connect CO : CoinsInfo m_coinsServer**

    G -> CO : CID_G2CO_LOGIN
    CO : 保存 std::map<Lint, GateInfo> m_gateInfo

## 7. Login - LN ##
    listen
        0.0.0.0:5031    outside
    connect
        CE  2001


**<1> LN connect CE : LSocketPtr m_socketCenter**

    CE : 没标记socket

# 客户端登陆 #

## 1.Client connect LN ##

    C->LN->CE
        CID_C2S_LOGIN_LOGIN
        CID_LN2CE_FROM_LOGINSERVER
            if 新用户
                http get : https://api.weixin.qq.com/sns/userinfo?access_token=27_8e8DF0x6Wh1frPPxvTWhxVdwpVJKudQDCL5i1FhviPOy1H2uQwp_cbBrN42A01rEsHqA8L58ivaLWKuA34RDvxvoZiGQ6rKw4-k5w18o1oM&openid=ozm8_0imzX_Q-o7LuEyCcgA8_yQc
                MySQL : INSERT INTO user (id,openId,nike,sex,token,channel,provice,city,country,headImageUrl,unionId,numsCard1,numsCard2,numsCard3,lastLoginTime,regTime,new,gm,totalCardNum,totalPlayNum, masterScore, masterScoreUsed, masterId, masterCode) VALUES ('11028880','ozm8_0imzX_Q-o7LuEyCcgA8_yQc','馃惏','1','6f56725852ca742bc148d855eaa6117c','yyb','','','','http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiaNDFLxRqCJXmtnpYSCSf91xp1diaJJ6U9rN9bCibBaNBRKWafiahRzRN0f3XSick0MVHpfIb6YniaLSw/132','oXCvywkVWWMX0DlEKGvo2eIHlwtQ','0','0','0','1573808800','1573808800','0','0','0','0','0','0','0','')
            else
                ...
    CE->LM : CID_CE2LM_USER_LOGIN
    CE->LN->C
            CID_S2C_LOGIN_LOGIN
            CID_CE2LN_TO_LOGINSERVER
    C : close

## 2.Client connect G ##

    //登录LM，将CE、LDB中数据同步到LM
    //登录CO，将LM中数据同步到CO
    //之后，读数据直接从LM或CO程序内存中读，写数据往LM、CO中写，往CE、LDB中写不用考虑返回值
    //读写都是非阻塞的

    C->G, G->C
        CID_C2S_LOGIN_GATE_1
        CID_S2C_LOGIN_GATE_1
    C->G->LM
        CID_C2S_LOGIN_GATE_2
        CID_G2LM_G2CO_G2L_USER_MSG
            Redis : SADD MasterSys:BigRedPacket:318 11028880
            Redis : SADD MasterSys:BigPacketGiven 11028880
            Redis : EXISTS DailyTask:20191115:11028880:
            Redis : HMSET DailyTask:20191115:11028880: 1 0 10 0 11 0 2 0 3 0 4 0 5 0 6 0 7 0 8 0 9 0 daily_task_pos 1 2 3 4 5 6 7 8 9 10 11 money_tree_next_time 1573749000 money_tree_times 0 senior_task_flag 0
            Redis : EXPIRE DailyTask:20191115:11028880: 86400
            Redis : HGET master_new_relation oXCvywkVWWMX0DlEKGvo2eIHlwtQ
            Redis : HDEL master_new_relation oXCvywkVWWMX0DlEKGvo2eIHlwtQ

    //修改金币，修改用户属性字段
    LM->CE
        CID_CO2LM_LM2CE_MODIFY_USER_COINS
            MySQL : <INSERT INTO chargelog (userid, chargeCoins, chargeOper, operParam) VALUES ('11028880','6666','109','')>
            MySQL : <UPDATE user SET nike='馃惏',sex='1',provice='',phone='',city='',country='',headImageUrl='http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiaNDFLxRqCJXmtnpYSCSf91xp1diaJJ6U9rN9bCibBaNBRKWafiahRzRN0f3XSick0MVHpfIb6YniaLSw/132',numsCard1='0',numsCard2='0',numsCard3='0',lastLoginTime='1573808800',totalCardNum='0',new='0',gm='0',totalPlayNum='0',coins='6666',credits='0',ticket='0',diamond='0',isMove='0',totalScore='0',score='0',masterScore='0',masterScoreUsed='0',masterId='0',masterCode='',playCount_0='0', winCount_0='0',playCount_1='0', winCount_1='0',playCount_2='0', winCount_2='0',playCount_3='0', winCount_3='0',limitCountFlag='  ' WHERE unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>
    LM->G->C
        CID_S2C_ITEM_INFO
        CID_LM2G_CO2G_L2G_USER_MSG

    //修改用户属性字段
    LM->CE
        CID_LM2CE_MODIFY_USER_NEW
            MySQL : <UPDATE user SET nike='馃惏',sex='1',provice='',phone='',city='',country='',headImageUrl='http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiaNDFLxRqCJXmtnpYSCSf91xp1diaJJ6U9rN9bCibBaNBRKWafiahRzRN0f3XSick0MVHpfIb6YniaLSw/132',numsCard1='0',numsCard2='0',numsCard3='0',lastLoginTime='1573808800',totalCardNum='0',new='1',gm='0',totalPlayNum='0',coins='6666',credits='0',ticket='0',diamond='0',isMove='0',totalScore='0',score='0',masterScore='0',masterScoreUsed='0',masterId='0',masterCode='',playCount_0='0', winCount_0='0',playCount_1='0', winCount_1='0',playCount_2='0', winCount_2='0',playCount_3='0', winCount_3='0',limitCountFlag='  ' WHERE unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>
    LM->CE
        CID_LM2CE_SAVE_CITY
            MySQL : <update user set city='0' where unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>

    //修改用户属性字段
    LM :
        Redis : SPOP MasterSys:MasterCodePool
        Redis : HSET MasterSys:Code2Id ns1c76 11028880
    LM->CE
        CID_LM2CE_MASTER_CODE_CREATE
            MySQL : <update user set masterCode='ns1c76' where unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>

    //回复
    LM->G->C
        CID_S2C_LOGIN_GATE_2

    LM->G->C
        CID_S2C_SEND_NEW_PLAYER_GIFT
        CID_S2C_ITEM_INFO

    //登录CO
    LM->CO
        CID_LM2CO_USER_LOGIN

    //登录LDB
    LM->LDB, LDB->LM->G->C
        CID_LM2LDB_USER_LOGIN
        CID_S2C_ACTIVITY_INFO
        CID_LDB2LM_USER_MSG1
    LM->LDB, LDB->LM->G->C
        CID_LM2LDB_QUERY_UNREAD_MAIL_COUNT
            MySQL : select count(cMailId) from mail where cReceiverId = 11028880 and cReadStatus = 0
        CID_S2C_QUERY_UNREAD_MAIL_COUNT
        CID_LDB2LM_USER_MSG2

    LM->G->C
        CID_S2C_PREN_FIRST_RED_PACKET_NOTIFY

    //记牌器
    LM->CE
        CID_LM2CE_CHANGE_LIMIT_FLAG
            MySQL : <update user set limitCountFlag=' 1 ' where unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>
    LM->CE、LM->CO
        CID_LM2CE_LM2CO_CO2LM_CO2L_SERVER_SYNC_ATTRI
            MySQL : <update user set cardRecordTime=1574413601 where unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>
    LM->LDB
        CID_LM2LDB_CREATE_MAIL
            MySQL : call ProcCreatePersonalMail(2, 0, 0, '系统邮件', 11028880, '【记牌器】限时体验', '斗地主【记牌器】限时体验，免费使用7天。\n您当前【记牌器】有效期截止至2019/11/22 17:06:41')
    LM->G->C
        CID_S2C_NEW_MAIL_NOTIFY

    //跑马灯
    LM->G->C
        CID_S2C_HORSE_INFO
        CID_S2C_HORSE_INFO

    //登录CE
    LM->CE
        CID_LM2CE_USER_LOGIN
            MySQL : <UPDATE user SET nike='馃惏',sex='1',provice='',city='0',country='',headImageUrl='http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiaNDFLxRqCJXmtnpYSCSf91xp1diaJJ6U9rN9bCibBaNBRKWafiahRzRN0f3XSick0MVHpfIb6YniaLSw/132',numsCard1='0',numsCard2='0',numsCard3='0',lastLoginTime='1573808801',new='1',totalCardNum='0',totalPlayNum='0',coins='6666',credits='0',ticket='0',masterScore='0',masterScoreUsed='0',masterId='0',masterCode='ns1c76',playCount_0='0', winCount_0='0',playCount_1='0', winCount_1='0',playCount_2='0', winCount_2='0',playCount_3='0', winCount_3='0',limitCountFlag=' 1 ' WHERE unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>

## Client 登录后，会自动发送很多查询请求 ##

    C->G, G->C per 5s
        CID_C2S_HEART
        CID_S2C_HEART

    C->G->LM，LM->G->C
        CID_C2S_GET_GLOBAL_DATA
            Redis : GET client_global_data
        CID_S2C_GET_GLOBAL_DATA

        CID_C2S_QUERY_LAIXIA_MOVE
        CID_S2C_QUERY_LAIXIA_MOVE

        CID_C2S_TASK_QUERY
        CID_S2C_TASK_QUERY

        CID_C2S_GAME_SHARE
            Redis : HGET game_share:20191115 11028880
        CID_S2C_SEND_GAME_SHARE_INFO

## 新人红包 ##

    C->G->LM
        CID_C2S_OPEN_RED_PACKET
            Redis : SREM MasterSys:BigRedPacket:318 11028880
    LM->G->C
        CID_S2C_ITEM_INFO
    LM->CE
        CID_LM2CE_LM2CO_CO2LM_CO2L_SERVER_SYNC_ATTRI
            MySQL : <INSERT INTO scorelog (userId, num, oper, operString) VALUES ('11028880','1000','13008','')>
            MySQL : <UPDATE user SET nike='馃惏',sex='1',provice='',phone='',city='0',country='',headImageUrl='http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiaNDFLxRqCJXmtnpYSCSf91xp1diaJJ6U9rN9bCibBaNBRKWafiahRzRN0f3XSick0MVHpfIb6YniaLSw/132',numsCard1='0',numsCard2='0',numsCard3='0',lastLoginTime='1573808801',totalCardNum='0',new='1',gm='0',totalPlayNum='0',coins='6666',credits='0',ticket='0',diamond='0',isMove='0',totalScore='1000',score='1000',masterScore='0',masterScoreUsed='0',masterId='0',masterCode='ns1c76',playCount_0='0', winCount_0='0',playCount_1='0', winCount_1='0',playCount_2='0', winCount_2='0',playCount_3='0', winCount_3='0',limitCountFlag=' 1 ' WHERE unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>
    LM->CO
        CID_LM2CE_LM2CO_CO2LM_CO2L_SERVER_SYNC_ATTRI

    //回复
    LM->G->C
        CID_S2C_OPEN_RED_PACKET

## “10元可提现，现在取游戏赚取第一桶金吧！” 点击 “开始游戏” ##

    C->G->LM->LDB，LDB->LM->G->C
        CID_C2S_LM2LDB_GET_SEVEN_DAY_INFO //C2G、LM2LDB（通用）
        CID_S2C_SEND_SEVEN_DAY_INFO
        CID_LDB2LM_USER_MSG1

        CID_C2S_LM2LDB_MOVE_SHARE_QUERY //C2G、LM2LDB（通用）
            Redis : SISMEMBER app_move_share_user 11028880
        CID_S2C_MOVE_SHARE_QUERY
        CID_LDB2LM_USER_MSG1

    C->G->LM, LM->G->C
        CID_C2S_GET_SAVE_LIFE_INFO
            Redis : HGET save-life-coins:20191115 11028880
        CID_S2C_SEND_SAVE_LIFE_INFO

## 每日签到 “领取” ##

    C->G->LM->LDB，LDB->LM->G->C
        CID_C2S_LM2LDB_GET_SEVEN_DAY_INFO //C2G、LM2LDB（通用）
        CID_S2C_SEND_SEVEN_DAY_INFO
        CID_LDB2LM_USER_MSG1

    C->G->LM->LDB，LDB->LM->G->C
        CID_C2S_LM2LDB_SEVEN_DAY_SIGN_IN //C2G、LM2LDB（通用）
        CID_S2C_SEND_SEVEN_DAY_INFO
        *
        CID_S2C_SEVEN_DAY_SIGN_IN
            MySQL : INSERT INTO seven_day_log (userId, day, lastTime) VALUES (11028880,1,1573809141) ON DUPLICATE KEY UPDATE day=1,lastTime=1573809141;
    *LDB->LM
        CID_LDB2LM_ADD_MONEY
    LM->CE
        CID_CO2LM_LM2CE_MODIFY_USER_COINS
            MySQL : <INSERT INTO chargelog (userid, chargeCoins, chargeOper, operParam) VALUES ('11028880','100','110','')>
            MySQL : <UPDATE user SET nike='馃惏',sex='1',provice='',phone='',city='0',country='',headImageUrl='http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiaNDFLxRqCJXmtnpYSCSf91xp1diaJJ6U9rN9bCibBaNBRKWafiahRzRN0f3XSick0MVHpfIb6YniaLSw/132',numsCard1='0',numsCard2='0',numsCard3='0',lastLoginTime='1573808801',totalCardNum='0',new='1',gm='0',totalPlayNum='0',coins='6766',credits='0',ticket='0',diamond='0',isMove='0',totalScore='1000',score='1000',masterScore='0',masterScoreUsed='0',masterId='0',masterCode='ns1c76',playCount_0='0', winCount_0='0',playCount_1='0', winCount_1='0',playCount_2='0', winCount_2='0',playCount_3='0', winCount_3='0',limitCountFlag=' 1 ' WHERE unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>
    LM->G->C
        CID_S2C_ITEM_INFO

## 退出登录 ##

    C : close
    G->LM, G->CO
        CID_G2LM_G2CO_USER_OUT_MSG
    LM->CE
        CID_LM2CE_USER_LOGOUT
            MySQL : <UPDATE user SET nike='馃惏',sex='1',provice='',phone='',city='0',country='',headImageUrl='http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiaNDFLxRqCJXmtnpYSCSf91xp1diaJJ6U9rN9bCibBaNBRKWafiahRzRN0f3XSick0MVHpfIb6YniaLSw/132',numsCard1='0',numsCard2='0',numsCard3='0',lastLoginTime='1573808801',totalCardNum='0',new='1',gm='0',totalPlayNum='0',coins='6766',credits='0',ticket='0',diamond='0',isMove='0',totalScore='1000',score='1000',masterScore='0',masterScoreUsed='0',masterId='0',masterCode='ns1c76',playCount_0='0', winCount_0='0',playCount_1='0', winCount_1='0',playCount_2='0', winCount_2='0',playCount_3='0', winCount_3='0',limitCountFlag=' 1 ' WHERE unionId='oXCvywkVWWMX0DlEKGvo2eIHlwtQ'>

# 血流成河 金币场 游戏逻辑 #

## 点击 “血流成河” ##

    C->G->LM, LM->G->C
        CID_C2S_GAME_SHARE
            Redis : HGET game_share:20191116 13914040
        CID_S2C_SEND_GAME_SHARE_INFO

## 点击 “初级场” ##

    C->G->LM
        CID_C2S_ENTER_COINS_ROOM
    LM->CO
        CID_LM2CO_ENTER_COIN_DESK

    //debug模式下，可以设置金币
    CO->LM，LM->CE
        CID_CO2LM_LM2CE_MODIFY_USER_COINS
            MySQL : <INSERT INTO chargelog (userid, chargeCoins, chargeOper, operParam) VALUES ('13914040','0','100','')>
            MySQL : <UPDATE user SET nike='游客',sex='1',provice='',phone='',city='0',country='',headImageUrl='',numsCard1='0',numsCard2='0',numsCard3='0',lastLoginTime='1573906381',totalCardNum='0',new='1',gm='0',totalPlayNum='0',coins='6766',credits='0',ticket='0',diamond='0',isMove='0',totalScore='1000',score='1000',masterScore='0',masterScoreUsed='0',masterId='0',masterCode='tj3z68',playCount_0='0', winCount_0='0',playCount_1='0', winCount_1='0',playCount_2='0', winCount_2='0',playCount_3='0', winCount_3='0',limitCountFlag=' 1 ' WHERE unionId='test7696355'>
    LM->G->C
        CID_S2C_ITEM_INFO

    CO->G->C
        CID_S2C_INTO_DESK
        CID_S2C_ADD_USER //两人及以上的时候，给自己发送桌内其他人信息，给桌内其他人发送自己信息。第2个人进入的时候一共发2条，第3个人进入的时候一共发4条，第4个人进入的时候一共发6条。
        CID_S2C_ENTER_COINS_ROOM

    //修改玩家状态 LGU_STATE_COIN
    CO->LM
        CID_CO2LM_L2LM_L2CO_MODIFY_USER_STATE
    CO->G
        CID_CO2G_L2G_MODIFY_USER_STATE

    //第一个人进入的时候
    CO->LM，LM->CO
        CID_CO2LM_FREE_DESK_REQ
        CID_LM2CO_FREE_DESK_REPLY


    ///点击 “退出游戏”（未匹配成功时）
    C->G->CO, CO->G->C
        CID_C2S_LEAVE_ROOM
        CID_S2C_LEAVE_ROOM

    //修改玩家状态 LGU_STATE_CENTER //LM
    CO->LM
        CID_CO2LM_L2LM_L2CO_MODIFY_USER_STATE
    CO->G
        CID_CO2G_L2G_MODIFY_USER_STATE


    ///匹配成功
    CO->L
        CID_CO2L_CREATE_COIN_DESK

    广播 修改玩家状态 LGU_STATE_COINDESK
    L->G
        CID_CO2G_L2G_MODIFY_USER_STATE
    L->LM, L->CO
        CID_CO2LM_L2LM_L2CO_MODIFY_USER_STATE

    L->LM
        CID_L2LM_ON_GAME_START

    广播
    LM : Redis : HINCRBY ttmj:playCount:13914040 103_1 1
         Redis : SET ttmj:lastPlayReward:13914040 1/5
    LM->G->C
        CID_S2C_PLAY_COUNT_NOTIFY

    广播
    LM->G->C
        CID_S2C_VIP_INFO
        CID_S2C_START
        CID_S2C_BROADCAST_MONEY_CHANGE

    L->CO
        CID_L2CO_CREATE_COIN_DESK_RET

    广播
    CO->LM, LM->CE
        CID_CO2LM_LM2CE_MODIFY_USER_COINS
            CO : Redis : HINCRBY record:20191116:106 10301 350
            CE : MySQL : <INSERT INTO chargelog (userid, chargeCoins, chargeOper, operParam) VALUES ('13914040','-350','106','')>
                 MySQL : <UPDATE user SET nike='游客',sex='1',provice='',phone='',city='0',country='',headImageUrl='',numsCard1='0',numsCard2='0',numsCard3='0',lastLoginTime='1573906381',totalCardNum='0',new='1',gm='0',totalPlayNum='0',coins='6416',credits='0',ticket='0',diamond='0',isMove='0',totalScore='1000',score='1000',masterScore='0',masterScoreUsed='0',masterId='0',masterCode='tj3z68',playCount_0='0', winCount_0='0',playCount_1='0', winCount_1='0',playCount_2='0', winCount_2='0',playCount_3='0', winCount_3='0',limitCountFlag=' 1 ' WHERE unionId='test7696355'>
    LM->G->C
        CID_S2C_ITEM_INFO
    CO->G->C
        CID_S2C_ATTRI64_UPDATE
    CO->LM, LM->CE
        CID_LM2CE_LM2CO_CO2LM_CO2L_SERVER_SYNC_ATTRI
            MySQL : <update user set userExp=35 where unionId='test7696355'>
    CO->L
        CID_LM2CE_LM2CO_CO2LM_CO2L_SERVER_SYNC_ATTRI
    CO->LM
        CID_CO2LM_PREN_COST


    ///4个玩家自动玩（当超时时，Logic会模拟客户端应发的消息）

    /////////状态 DESK_PLAY_START_WAIT 游戏前等待超时
    只是增加计数

    达到4人时
    广播
        CID_S2C_USER_CHANGE //换牌开始，桌子状态切换为 DESK_PLAY_CHANGE


    /////////状态 DESK_PLAY_CHANGE 换牌超时
        CID_S2C_USER_AIOPER
    模拟
        CID_C2S_USER_CHANGE //玩家要换的牌
    广播
        CID_S2C_USER_CHANGE //玩家换的牌

    达到4人时
    广播
        CID_S2C_USER_CHANGERESULT //换牌结果
    广播
        CID_S2C_USER_DINGQUE_START //定缺开始，桌子状态切换为 DESK_PLAY_DINGQUE


    /////////状态 DESK_PLAY_DINGQUE 定缺超时
        CID_S2C_USER_AIOPER
    模拟
        CID_C2S_USER_DINGQUE //玩家要定的缺
    广播（前3个玩家时）
        CID_S2C_USER_DINGQUE_START //玩家定的缺

    达到4人时
    广播
        CID_S2C_USER_DINGQUE_COMPLETE //定缺结果
    广播（摸牌位置是自己时，才明牌发送摸到的牌、思考信息）
        CID_S2C_GET_CARD //玩家摸的牌，出牌开始，桌子状态切换为 DESK_PLAY_GET_CARD


    /////////状态 DESK_PLAY_GET_CARD 出牌超时
        CID_S2C_USER_AIOPER
    模拟
        CID_C2S_SET_TUOGUAN //玩家要设置托管
    广播
        CID_S2C_BROADCAST_TUOGUAN //玩家设置托管

    模拟
        CID_C2S_PLAY_CARD //玩家要出的牌
    只发给自己
        CID_S2C_PLAY_CARD_RESPONSE //前端要求加一个出牌反馈消息
    广播
        CID_S2C_PLAY_CARD //玩家出的牌

    当有思考信息时，广播
        CID_S2C_USER_THINK //玩家思考信息，思考开始，桌子状态切换为 DESK_PLAY_THINK_CARD
    没有思考信息时，广播
        CID_S2C_GET_CARD //下一个玩家摸的牌，出牌开始


    /////////状态 DESK_PLAY_THINK_CARD
    CID_S2C_USER_AIOPER
    模拟
        CID_C2S_SET_TUOGUAN //玩家要设置托管
    广播
        CID_S2C_BROADCAST_TUOGUAN //玩家设置托管

    模拟
        CID_C2S_USER_OPER //玩家的思考操作，默认是 THINK_OPERATOR_NULL “过”
    广播
        CID_S2C_GET_CARD //下一个玩家摸的牌，出牌开始，桌子状态切换为 DESK_PLAY_GET_CARD


    ///游戏结束
    L->LDB
        CID_L2LDB_VIDEO_SAVE
            MySQL : INSERT INTO video (Id,Time,UserId1,UserId2,UserId3,UserId4,Zhuang,DeskId,MaxCircle,CurCircle,Score1,Score2,Score3,Score4,Flag,Data, PlayType) VALUES ('100515739187092','1573910965','13914040','14063949','14513531','14081675','0','917586','1','0','0','0','0','0','103','1,2,1,6,2,2,2,3,2,5,2,8,2,8,3,1,3,2,3,2,3,3,3,4,3,5,3,8;1,3,1,5,2,2,2,4,2,7,2,8,2,9,3,1,3,2,3,6,3,6,3,7,3,9;1,2,1,3,2,3,2,3,2,4,2,5,2,6,2,7,3,3,3,5,3,7,3,7,3,9;1,1,1,1,1,2,1,2,1,5,2,1,2,2,2,6,2,7,2,7,2,9,3,3,3,9;0,58,2|2,2|3,2|5,2|2,2|4,2|7;1,58,2|2,2|4,2|7,3|3,3|5,3|7;2,58,3|3,3|5,3|7,1|1,1|1,1|2;3,58,1|1,1|1,1|2,2|2,2|3,2|5;0,57,1|1;1,57,1|1;2,57,3|1;3,57,1|1;0,2,1|2;2,21,12|5;2,22,0|0;1,1,1|8;1,2,1|3;2,1,1|6;2,2,3|7;1,21,37|5;1,22,0|0;3,1,1|7;3,2,1|2;2,21,12|5;2,22,0|0;0,1,3|7;0,2,1|6;1,1,1|5;1,2,1|5;2,1,1|1;2,2,3|9;3,1,3|1;3,2,1|5;0,1,3|9;0,2,3|9;1,1,3|3;1,2,1|5;2,1,1|4;2,2,1|4;3,1,2|9;3,2,1|7;0,1,1|6;0,2,1|6;1,1,3|8;1,2,1|8;2,1,2|6;2,2,2|6;3,1,2|5;3,2,2|5;0,1,2|6;0,2,2|6;1,1,3|2;1,2,3|2;0,21,32|5;0,22,0|0;2,1,3|6;2,2,3|6;1,21,36|5;
    广播 L->G->C
        CID_S2C_GAMEREULT //单局结算信息
    广播 L->LM LM->CE, L->CO
        CID_L2LM_L2CO_LM2CE_ADD_PLAY_COUNT //同步修改总局数和获胜局数
            MySQL : <update user set totalPlayNum = 1, playCount_0 = 1, winCount_0 = 0 where unionId = 'test7696355'>
    L->LDB
        CID_L2LDB_ROOM_SAVE
            MySQL : INSERT INTO log (Id,Time,Pos1,Pos2,Pos3,Pos4,Flag,BaseScore,DeskId,MaxCircle,CurCircle,Pass,Score1,Score2,Score3,Score4,Reset,checkTing1,checkTing2,checkTing3,checkTing4,Data,PlayType) VALUES ('100515739109652','1573910965','13914040','14063949','14513531','14081675','10301','1','917586','1','1','','0','0','0','0','0','0','0','0','0','0,0,0,0;0,0,0,0;0,0,0,0;0;1573918709;100515739187092;0,0,0,0;0,0,0,0;0,0,0,0;0,0,0,0;0,0,0,0;0,0,0,0;0,0,0,0;|','23;30;27;34;20;28;')
    广播 L->G->C
        CID_S2C_GAME_OVER //游戏结束
    广播 L->LM
        CID_L2LM_UPDATE_TASK
            Redis : HSET DailyTask:20191116:13914040: 3 1
            Redis : HSET DailyTask:20191116:13914040: 5 1
            Redis : HSET DailyTask:20191116:13914040: 7 0
    L->CO
        CID_L2CO_END_COIN_DESK //结算
            Redis : GET ttmj:lastPlayReward:13914040
            Redis : LPUSH gameRecord:13914040 {"coin":0,"reward":"1/5","time":1573918709,"type":103001}
            Redis : LTRIM gameRecord:13914040 0 49
            Redis : EXPIRE gameRecord:13914040 604800

    Redis : INCRBY ttmj:totalPlayCount:18216 1
    Redis : EXPIRE ttmj:totalPlayCount:18216 86400
    Redis : SET ttmj:RoundInfo:1 {"level":1,"score":{"游客":{"multi":1,"win":0}},"type":103}
    L->LM
        CID_L2LM_ON_GAME_OVER
            Redis : GET ttmj:RoundInfo:1

    广播 L->G->C
        CID_S2C_VIP_END

    广播 修改玩家状态 LGU_STATE_COIN
    L->G
        CID_CO2G_L2G_MODIFY_USER_STATE
    L->LM, L->CO
        CID_CO2LM_L2LM_L2CO_MODIFY_USER_STATE

## 点击 “下一局” ##

    C->G->CO, CO->G->C
        CID_C2S_GO_ON_COINS_ROOM
        CID_S2C_INTO_DESK
        CID_S2C_ADD_USER
        CID_S2C_GO_ON_COINS_ROOM

    广播
    L->LM
        CID_L2LM_UPDATE_TASK
            Redis : HSET DailyTask:20191116:13914040: 3 2
            Redis : HSET DailyTask:20191116:13914040: 5 2
            Redis : HSET DailyTask:20191116:13914040: 7 0
    LM->G->C
        CID_S2C_TASK_REWARD_NOTIFY //通知任务完成

# 血流成河 好友房 游戏逻辑 #

## 点击 “创建房间”->"确认" ##

    C->G-LM
        CID_C2S_CREATE_FRIEND_DESK
    LM->L
        CID_LM2L_CREATE_FRIEND_DESK
    L->G->C
        CID_S2C_INTO_DESK
        CID_S2C_CREATE_FRINED_DESK_RET
    修改玩家状态 LGU_STATE_DESK
    L->G
        CID_CO2G_L2G_MODIFY_USER_STATE
    L->LM, L->CO
        CID_CO2LM_L2LM_L2CO_MODIFY_USER_STATE

## 点击 “加入房间”，输入房间号 ##

    C->G->LM
        CID_C2S_ADD_FRIEND_DESK
    LM->L
        CID_LM2L_ADD_FRIEND_DESK
    L->G->C
        CID_S2C_INTO_DESK
        CID_S2C_ADD_USER *2, *4, *6
    {   CID_S2C_ADD_FRIEND_DESK_RET
    修改玩家状态 LGU_STATE_DESK
    L->G
        CID_CO2G_L2G_MODIFY_USER_STATE
    L->LM, L->CO
        CID_CO2LM_L2LM_L2CO_MODIFY_USER_STATE
    } 第4个人时，发送这几个消息前，会发送游戏开始的一系列消息，同金币场

    ///游戏结束
    L->LDB
        CID_L2LDB_VIDEO_SAVE
            MySQL : INSERT INTO video (Id,Time,UserId1,UserId2,UserId3,UserId4,Zhuang,DeskId,MaxCircle,CurCircle,Score1,Score2,Score3,Score4,Flag,Data, PlayType) VALUES ('100515740685882','1574068236','13914040','14063949','14513531','14081675','0','714683','1','0','0','0','0','0','113','1,1,1,1,1,2,1,5,1,7,2,3,2,4,2,5,3,3,3,4,3,4,3,4,3,6,3,9;1,2,1,2,1,4,1,6,1,6,1,7,1,8,1,8,1,9,2,5,2,7,3,3,3,8;1,1,1,1,1,3,1,4,1,7,2,1,2,7,2,9,3,2,3,2,3,4,3,6,3,8;1,2,1,5,1,6,1,6,2,3,2,4,2,8,2,8,2,9,3,1,3,2,3,5,3,9;0,58,2|3,2|4,2|5,3|1,3|2,3|5;1,58,1|2,1|2,1|4,2|3,2|4,2|5;2,58,2|1,2|7,2|9,1|2,1|2,1|4;3,58,3|1,3|2,3|5,2|1,2|7,2|9;0,57,2|1;1,57,3|1;2,57,2|1;3,57,3|1;0,2,3|9;1,1,1|9;1,2,3|3;2,1,1|3;2,2,1|3;3,1,2|2;3,2,3|9;0,1,3|1;0,2,3|1;1,1,3|8;1,2,3|8;2,1,2|3;2,2,2|3;3,1,2|4;3,2,2|4;0,1,3|6;0,2,3|6;1,1,2|2;1,2,3|8;2,1,1|3;2,2,1|3;3,1,2|7;3,2,2|7;0,1,1|4;0,2,1|4;2,21,14|5;2,22,0|0;1,1,3|1;1,2,3|1;2,1,2|5;2,2,2|5;1,21,25|5;1,22,0|0;3,1,2|5;3,2,2|5;1,21,25|5;1,22,0|0;0,1,2|1;0,2,2|1;1,1,1|8;1,2,1|8;2,1,1|5;2,2,1|5;3,1,2|1;3,2,2|1;0,1,1|7;0,2,1
    广播 L->LM LM->CE, L->CO
        CID_L2LM_L2CO_LM2CE_ADD_PLAY_COUNT //同步修改总局数和获胜局数
            MySQL : <update user set totalPlayNum = 4, playCount_1 = 1, winCount_1 = 0 where unionId = 'test7164588'>
    L->LDB
        CID_L2LDB_ROOM_SAVE
            MySQL : INSERT INTO log (Id,Time,Pos1,Pos2,Pos3,Pos4,Flag,BaseScore,DeskId,MaxCircle,CurCircle,Pass,Score1,Score2,Score3,Score4,Reset,checkTing1,checkTing2,checkTing3,checkTing4,Data,PlayType) VALUES ('100515740685882','1574068588','13914040','14063949','14513531','14081675','11310','300','714683','1','1','','0','0','0','0','0','0','0','0','0','0,0,0,0;0,0,0,0;0,0,0,0;0;1574068588;100515740685882;0,0,0,0;0,0,0,0;0,0,0,0;0,0,0,0;0,0,0,0;0,0,0,0;0,0,0,0;|','23;30;27;34;20;28;')
    广播 L->G->C
        CID_S2C_GAME_OVER //游戏结束
    广播 L->LM
        CID_L2LM_UPDATE_TASK
            Redis : HSET DailyTask:20191118:13914040: 7 0
    L->CO
        CID_L2CO_SEND_MONEY
    广播 CO->LM
        CID_CO2LM_PREN_COST
    CO :
        Redis : GET ttmj:lastPlayReward:13914040
        Redis : LPUSH gameRecord:13914040 {"coin":0,"reward":"1/5","time":1574068588,"type":11310}
        Redis : LTRIM gameRecord:13914040 0 49
        Redis : EXPIRE gameRecord:13914040 604800
    Redis : HINCRBY record:20191118:106 11301 1600

    ///空闲超过一小时，好友房自动解散
    广播 L->G->C
        CID_S2C_DEL_USER *4 *3 *2 *1
    广播 修改玩家状态 LGU_STATE_CENTER //LM
    L->G
        CID_CO2G_L2G_MODIFY_USER_STATE
    L->LM, L->CO
        CID_CO2LM_L2LM_L2CO_MODIFY_USER_STATE

    L->LM
        CID_L2LM_RECYLE_DESKID

## 下一局 ##

    C->G->L
        CID_C2S_GO_ON_FRIEND_DESK
    广播 L->G->C
        CID_S2C_READY

## 离开好友房 ##

    C->G->L
        CID_C2S_LEAVE_FRIEND_DESK
    L->G->C
        CID_S2C_LEAVE_FRIEND_DESK_RET
    广播
        CID_S2C_DEL_USER
    修改玩家状态 LGU_STATE_CENTER
    L->G
        CID_CO2G_L2G_MODIFY_USER_STATE
    L->LM, L->CO
        CID_CO2LM_L2LM_L2CO_MODIFY_USER_STATE

## 点击 “确认 房间将解散，您是否退出” ##

    逻辑同 离开好友房
    L->LM
        CID_L2LM_RECYLE_DESKID

# 游戏逻辑 #

    Gate
        void OutsideNet::RecvMsgPack(LBuffPtr recv, LSocketPtr s, bool bIsFromInternal)
            gWork.Push(pConvertMsg);
        void Work::Run()
            HanderMsg(msg);
                _doWithClientMessage((LMsgConvertClientMsg*)msg);
                    _doWithClientOtherMsg(pMsg->msgEntity, pMsg->msgOriginData);
                        LMsgG2LUserMsg msgG2L;
                        if (_isOnlySendToLogicManager(msgEntity->m_msgId))
                            m_logicManager->Send(msgG2L.GetSendBuff());

    LogicM
        void InsideNet::RecvMsgPack(LBuffPtr recv, LSocketPtr s, bool bIsFromInternal)
            gWork.Push(pMsg);
                gUserMessageMsg.handlerMessage(pUserMsg->m_strUUID, pUserMsg); //命名为Push更合理
                    m_userMessage[iIndex].Push(pMsg);
        void CUserMessage::Run()
            HanderMsg(msg);
                HanderUserMsg((LMsgG2LUserMsg*)msg);
                    safeUser->getResource()->HanderMsg(msg->m_userMsg);
                        HanderUserEnterCoinsDesk((LMsgC2SEnterCoinsDesk*)msg);
                            LMsgS2CEnterCoinsDeskRet        创建房间结果
                            LMsgLMG2CNEnterCoinDesk         主要在LMsgC2SEnterCoinsDesk的基础上加入m_userData、地址
                            gWork.SendMessageToCoinsServer( send );
                                m_coinsServer.m_sp->Send(msg.GetSendBuff());

    Coin
        void CUserMessage::Run()
            HanderMsg(msg);
                HanderEnterCoinDesk((LMsgLMG2CNEnterCoinDesk*)msg);
                    LMsgS2CEnterCoinsDeskRet                创建房间结果

                    UserPtr user(new User(msg->m_usert, msg->m_gateId, msg->m_ip)); //创建新用户
                    gUserManager.AddUser(user); //添加玩家

                    Lint ret = gDeskManager.UserEnterDesk( user->GetUserDataId(), (GameType)msg->m_state, msg->m_playType, msg->m_robotNum, msg->m_cardValue, (GameDifficulty) msg->m_difficulty );
                        //先在m_deskWaitList中查找同类型、难度的桌子，没有的话创建一个桌子
                        if ( desk->OnUserInRoom( user ) )
                            DeskUser deskuser(user->GetUserDataId(), pos);
                            user->SetDeskRunID( m_runID ); //标记用户进入桌子
                            LMsgS2CIntoDesk //用户进入桌子，仅自己显示自己进入
                            LMsgS2CDeskAddUser //广播：自己显示之前进入的人、之前进入的人显示自己
                            m_users.push_back( deskuser ); //加入玩家列表
                        //user带的机器人进入同一个桌子
                        gUserManager.AddUser( robot );
                        desk->OnUserInRoom( safeUser->getResource() );

        //自动匹配
        void DeskManager::Run()
            TickOneSecond();
                遍历m_deskWaitList
                Desk::CheckGameStart
                    桌子上玩家都准备好，并且等一秒（要进行第二次遍历），才算准备好
                CoinsDesk desk = _getCoinsDesk( (*it)->m_gameType );
                    在m_freeDeskList中取出桌子号CoinsDesk
                    如果桌子号中的m_logicID有问题，向LogicM发送LMsgCN2LMGRecycleDesk，回收这个桌子
                LMsgCN2LCreateCoinDesk 发给Logic，开桌消息
                桌子DeskPtr从m_deskWaitList转移到m_deskPlayList

    Logic //单线程
        void Work::Run()
            HanderMsg(msg);
                HanderLMGCreateCoinDesk((LMsgCN2LCreateCoinDesk*)msg);
                    Lint nErrorCode = gRoomVip.CreateVipCoinDesk(msg, userArray);
                        Desk* desk = GetFreeDesk(pMsg->m_deskId, (GameType)pMsg->m_state); //从m_deskMap中找DESK_FREE的桌子，改成DESK_WAIT；或创建新桌子
                        desk->OnUserInRoom(pUsers); //用户进入桌子
                            CheckGameStart();
                                所有人都准备好
                                mGameHandler->SetDeskPlay();
                                    DeakCard();
                                    扣除服务费，并不发给LM，只是本地修改 coins 和发给 client
                                    if 换三张
                                        m_desk->setDeskPlayState(DESK_PLAY_START_WAIT);
                                        m_desk->SetAutoPlay(i, true, m_desk->m_startWaitTime);
                                    else
                                        m_desk->setDeskPlayState(DESK_PLAY_DINGQUE);
                                        m_desk->SetAutoPlay(i, true, m_desk->m_dingQueTime);
                                        sendStartDingQueMsg();
                    SendToCoinsServer(ret); //Logic上创建桌子结果发给Coin，在Coin上扣服务费等

    Coin
        void InsideNet::RecvMsgPack(LBuffPtr recv, LSocketPtr s, bool bIsFromInternal)
            gWork.Push(pMsg);
                LRunnable::Push(msg);
        void Work::Run()
            HanderMsg(msg);
                HanderCreateCoinDeskRet((LMsgL2CNCreateCoinDeskRet*)msg);
                    gDeskManager.StartPlayDesk(msg->m_deskId);
                        //本地修改并发给其他服务器
                        user->DelCoinsCount(CoinCost, coinType);
                            LMsgCN2LMGModifyUserCoins 发给LogicM
                            LogicM再把 LMsgCN2LMGModifyUserCoins 发给Center，并把LMsgS2CItemInfo发给Client
                            Center把金币减少写入表chargelog、user
                        //写入redis
                        recordCoinToRedis(type, diffi, COINS_OPER_TYPE_GAME_START, CoinCost);
                            key = "record:20181127:106"
                            field = "10301"
                            value = "300"
                        //用户加经验
                        user->ChangeAttri64(PLAYER_ATTRI64_EXP, (Lint)ceil(CoinCost * gConfig.GetCoinToExpRate()));
                            本地修改
                            LMsgS2CAttri64Update发给Client
                            LMsgServerSyncAttri发给LogicM和Logic
                            LogicM再把LMsgServerSyncAttri发给Center
                                Center写入表user（调用存储过程proc_add_master_score）
                                Logic只更新自己本地
                        //师徒系统处理
                        LMsgCOS2LMPrenCost 发给LogicM

                        //考虑重构消息的发送
                            L->LM->CE
                            L->CO,
                            L->G

    Logic
        void Work::Run()
            Tick(cur);
                gRoomVip.Tick(cur);
                    void Desk::Tick(LTime& curr)
                        CheckAutoPlayCard(); //检查一个桌子上所有玩家

                            如果m_autoOutTime[i] = 10s。
                            0s时刻，消息发往client，需要1s才能到达。
                            1s时刻，client开始计时，11s时刻结束。
                            client 11s点击，12s才到server。
                            如果不延长server的outtime，server在10s时刻就会自动玩，这是不对的。
                            outtime += DIFFOPOUTTIME; //服务器的时间要长一些，增加2RTT

                            if (m_autoPlay[i] && cur.Secs() - m_autoPlayTime[i] > outtime)
                                m_autoPlay[i] = false; //下次if不通过，禁止在此处自动玩。在io_handler里也有。但DESK_PLAY_START_WAIT里有例外。考虑重构，只写在io_handler里。
                                mGameHandler->ProcessAutoPlay(i, m_user[i]);

                                    DESK_PLAY_START_WAIT //游戏前单纯等待
                                        m_startWaitNum++;
                                        if (m_startWaitNum >= DESK_USER_COUNT) //达到四个人
                                            LMsgS2CUserChange 广播开始换三张 *1
                                            //设置下一个状态，并设置当前时间和超时时间
                                            m_desk->setDeskPlayState(DESK_PLAY_CHANGE);
                                            m_desk->SetAutoPlay(i, true, m_desk->m_autoChangeOutTime);

                                    DESK_PLAY_CHANGE //换三张
                                        LMsgS2CUserAIOper THINK_OPERATOR_CHANGE 服务器AI操作
                                        LMsgC2SUserChange 找出牌数最少且大于等于三张的花色，取三张牌，模拟此消息
                                        m_desk->HanderUserChange(pUser, &msg);
                                            mGameHandler->HanderUserChange(pUser, msg);
                                                验证三张牌花色一致
                                                排序这三张牌后验证在手牌中存在这三张牌，然后m_change[pos][iCheckIndex] = m_handCard[pos][i];

                                                m_desk->SetAutoPlay(pos, false, 0); //和CheckAutoPlayCard里m_autoPlay[i] = false;可能重复
                                                LMsgS2CUserChange 广播，现在是明牌广播，和*1用同一个消息不太对
    
                                                if (changeCount == DESK_USER_COUNT)
                                                    OnUserChange();
                                                        //1 2顺时针，3 4对面交换，5，6逆时针
                                                        Lint dice = L_Rand(1, 6); //掷色子
                                                        LMsgS2CUserChangeResult //换后的牌，广播
                                                        本地手牌替换

                                                    m_desk->setDeskPlayState(DESK_PLAY_DINGQUE);
                                                    m_desk->SetAutoPlay(i, true, m_desk->m_dingQueTime);
                                                    sendStartDingQueMsg(); //广播开始定缺
                                                        LMsgS2CUserStartDingQue msgDingQue; 广播 //和确定定缺的消息用一个，不好
                                                        msgDingQue.m_timeout = m_desk->GetAutoPlayCountDown(i, m_desk->m_dingQueTime); //修正一些误差，感觉没必要

                                    DESK_PLAY_DINGQUE //定缺
                                        LMsgS2CUserAIOper THINK_OPERATOR_DINGQUE 服务器AI操作
                                        LMsgC2SUserDingQue 选择最少牌数的花色，模拟此消息
                                        m_desk->HanderUserDingQue(pUser, &msg);
                                            mGameHandler->HanderUserDingQue(pUser, msg);
                                                校验
                                                m_desk->SetAutoPlay(pos, false, 0);

                                                m_dingQueColor[pos] = msg->m_color;

                                                if (isAllUsersCompleteDingQue())
                                                    sendCompleteDingQueMsg
                                                        LMsgS2CUserCompleteDingQue //广播，玩家左上角出现定缺花色

                                                    // m_desk->setDeskPlayState(DESK_PLAY_GET_CARD);
                                                    SetPlayIng(m_curPos, false, false, true, true, true); //第一个人不需要摸牌
                                                        m_desk->setDeskPlayState(DESK_PLAY_GET_CARD); //等待出牌超时
                                                        不需要摸牌（实际是已经14张了），需要思考
                                                        //摸牌后自己的思考，检查胡、暗杠、补杠
                                                        m_thinkInfo[pos].m_thinkData = gCardMgrSc.CheckGetCardOperator(m_handCard[pos], m_pengCard[pos], m_angangCard[pos], m_gangCard[pos], needGetCard ? m_curGetCard : NULL, m_gameInfo);
                                                        //广播超时时间、要出牌的位置（、摸到的牌、思考数据）
                                                        LMsgS2COutCard 摸牌广播 msg.m_time = m_desk->GetOptTimeoutCfg(); 
                                                        m_desk->SetAutoPlay(pos, true, m_desk->m_autoPlayOutTime);
                                                else
                                                    sendStartDingQueMsg(); //广播某玩家定的缺
                                                        LMsgS2CUserStartDingQue //和开始定缺的消息用一个，不好

                                    DESK_PLAY_GET_CARD //出牌
                                        LMsgC2SUserPlay 是花猪的时候优先选缺一门花色的牌，否则选最后一张牌，模拟此消息
                                        LMsgS2CUserAIOper = LMsgC2SUserPlay
                                        设置托管
                                        m_desk->HanderUserPlayCard(pUser, &msg);
                                            mGameHandler->HanderUserPlayCard(pUser, msg);
                                                对LMsgC2SUserPlay进行验证
                                                LMsgS2CPlayCardResponse //验证回应，只发给自己
                                                LMsgS2CUserPlay sendMsg; //广播
                                                SetThinkIng //所有其他人想
                                                    //出牌后其他人的思考，检查胡、碰、明杠
                                                    m_thinkInfo[i].m_thinkData = gCardMgrSc.CheckOutCardOperator(m_handCard[i], m_pengCard[i], m_angangCard[i], m_gangCard[i], m_curOutCard, m_gameInfo);

                                                    if (think) //只要有一个人有思考数据，就进入DESK_PLAY_THINK_CARD状态
                                                        m_desk->setDeskPlayState(DESK_PLAY_THINK_CARD);
                                                        LMsgS2CThink think.m_time = m_desk->GetOptTimeoutCfg(); 广播超时时间、出的牌（、思考数据）
                                                        m_desk->SetAutoPlay(i, true, m_desk->m_autoPlayOutTime); //每个人设置成自动玩，有点问题，应该设置成有think数据的人自动玩

                                                    else
                                                        ThinkEnd //其他人没有思考结果
                                                            SetPlayIng(nextPos, true, false, true, true); //下一个玩家摸牌
                                                                m_desk->setDeskPlayState(DESK_PLAY_GET_CARD);
                                                                需要摸牌、思考
                                                                //这是下一个人摸牌后的自己的思考，检查胡、暗杠、补杠
                                                                m_thinkInfo[pos].m_thinkData = gCardMgrSc.CheckGetCardOperator(m_handCard[pos], m_pengCard[pos], m_angangCard[pos], m_gangCard[pos], needGetCard ? m_curGetCard : NULL, m_gameInfo);
                                                                LMsgS2COutCard 摸牌广播 msg.m_time = m_desk->GetOptTimeoutCfg(); //广播超时时间、要出牌的位置（、摸到的牌、思考数据）
                                                                m_desk->SetAutoPlay(pos, true, m_desk->m_autoPlayOutTime);

                                    DESK_PLAY_THINK_CARD
                                        // if (m_thinkInfo[pos].NeedThink()) 这里可能有问题，应该在之前设置成只有有think数据的人自动玩
                                        LMsgC2SUserOper THINK_OPERATOR_NULL 模拟的消息，默认选择“过”
                                        LMsgS2CUserAIOper = LMsgC2SUserOper 服务器AI操作
                                        托管
                                        m_desk->HanderUserOperCard(pUser, &msg);
                                            mGameHandler->HanderUserOperCard(pUser, msg);
                                                msg要与m_thinkInfo[pos]进行验证，若验证成功
                                                    m_thinkRet[pos] = m_thinkInfo[pos].m_thinkData[i];

                                                m_desk->SetAutoPlay(pos, false, 0);
                                                m_thinkInfo[pos].Reset();
                                                CheckThink
                                                    检查m_thinkRet和m_thinkInfo，判断是否还有新think。没有的话，就ThinkEnd

                                                    if (hu_new)
                                                        think = true
                                                    else if (!hu && (peng_new || gang_new || bu_new) //应该不可能
                                                        think = true
                                                    else
                                                        think = false

                                                    if (!think)
                                                        ThinkEnd
                                                            根据m_thinkRet找到胡、碰、明杠的位置
                                                            if (peng)
                                                                LMsgS2CUserOper 广播，胡、碰、明杠
                                                                gCardMgrSc.EraseCard(m_handCard[pengPos], m_curOutCard, 2);
                                                                m_pengCard[pengPos].push_back(m_curOutCard);
                                                                m_thinkRet[i].Clear();
                                                                m_curGetCard = NULL;
                                                                SetPlayIng(pengPos, false, false, false, false); //不摸牌 不think
                                                                    m_thinkInfo[pos].m_thinkData.clear();
                                                                    m_desk->setDeskPlayState(DESK_PLAY_GET_CARD); //等待出牌超时
                                                                    LMsgS2COutCard 摸牌广播 msg.m_time = m_desk->GetOptTimeoutCfg(); //广播超时时间、要出牌的位置，各玩家消息内容都一样
                                                                    m_desk->SetAutoPlay(pos, true, m_desk->m_autoPlayOutTime);
                                                                m_needGetCard = true;
                                                    else
                                                        桌子状态保持在think

## 简化逻辑： ##

    单线程————计算线程池线程数为1。此线程中含tick函数，每1ms检测一次期待的客户端消息是否超时。
    now-lastPlayTime[pos] > timeout
    若超时，由服务器模拟客户端消息，交给onMessage处理。
    单线程下：1 Room -> N Table -> 4N client
    Table划分[超时状态]，每个状态的超时时间+2s（矫正网络延迟）

**游戏前等待 Wait**

    lastPlayTime[pos] = INT_MAX
    ++waitCnt
    if waitCnt == 4
        deskState = Change3Cards
        i=0~3: lastPlayTime[i] = now
        广播 S2CChange3CardsBegin

**换三张 Change3Cards**

    S2CAI
    模拟收到C2SChange3Cards，交给onMessageChange3Cards处理
    onMessageChange3Cards
        lastPlayTime[pos] = INT_MAX
        保存数据，要换的三张牌
        广播 S2CChange3Cards
        ++changeCnt
        if changeCnt == 4
            玩家间换三张操作
            广播 S2CChange3CardsEnd

            deskState = Dingque
            i=0~3: lastPlayTime[i] = now
            广播 S2CDingqueBegin

**定缺 Dingque**

    S2CAI
    模拟收到C2SDingque，交给onMessageDingque处理
    onMessageDingque
        lastPlayTime[pos] = INT_MAX
        保存数据，定缺操作
        广播 S2CDingque
        ++dingqueCnt
        if dingqueCnt == 4
            广播 S2CDingqueEnd

            自己思考：庄家14张手牌，胡、暗杠

            deskState = PutCard
            lastPlayTime[pos] = now
            广播 S2CPutCardBegin 玩家的摸牌&思考数据

**出牌 PutCard**

    S2CAI
    模拟收到C2SPutCard，交给onMessagePutCard处理
    onMessagePutCard
        lastPlayTime[pos] = INT_MAX
        广播 S2CPutCard
        
        其他人思考: 胡、碰、明杠
        if 有思考数据
            deskState = OperCard
            有思考数据的每个位置i: lastPlayTime[i] = now
            广播 S2COperCardBegin 思考数据
        else
            下一个玩家摸牌
            下一个玩家思考：胡、暗杠、补杠

            lastPlayTime[nextPos] = now
            广播 S2CPutCardBegin 玩家的摸牌&思考数据

**鸣牌 OperCard**

    S2CAI
    模拟收到C2SOperCard，交给onMessageOperCard处理
    onMessageOperCard
        lastPlayTime[pos] = INT_MAX
        保存数据
        检测其他玩家是否还有合适的think数据
        if 没有
            集中鸣牌操作
            //胡
            //碰：等出牌
            //明杠：摸牌，思考，等出牌
        else
            返回
            //桌子保持在OperCard状态，继续处理剩余的鸣牌超时的玩家

## NOTE ##

    void DeskManager::Tick() :
    if ( m_deskWaitList.size() > 0 && m_LastRequestTime == 0 ) //这段代码至少3s才运行一次，申请（128*不包含类型数）个桌子
    {
    }

    void DeskManager::AddFreeDeskID( GameType type, const std::vector<CoinsDesk>& deskid )
    {
         m_LastRequestTime = 0; //桌子返回了，马上在Tick()里检查桌子是否够
    }

    Lint            m_waitSecond;       //由于时序问题，导致50号消息在31号消息之前到达，需要等待一秒
    bool Desk::CheckGameStart() //第一次并不开始，要等到一秒钟后的第二次，才往logic发开桌消息。这样确保31号先发出去。

    MSG_S_2_C_ADD_USER = 31,//桌子添加玩家
    MSG_S_2_C_START = 50,//服务器发送游戏开始消息


    外网 http

    用户注册 get 同步     https://api.weixin.qq.com/sns/userinfo?access_token=27_vpmMD__umTm9y61pF2BgyudQ8TtRUfEtMcKv0ZsOuLonHsPZGVj0UtKVG9bn3BfLnsbZe9tx8-SZJH1Ul3b3R9yRsb5KZ152K8QdR_6nPjA&openid=ozm8_0imzX_Q-o7LuEyCcgA8_yQc
    提现 post 异步  http://10.111.111.162/tx/accept
            {"msgId":1573184386,"openId":"ozm8_0imzX_Q-o7LuEyCcgA8_yQc","score":101000,"time":1573187482,"type":0,"userId":11929516}


    GM
    充金币
    http://192.168.43.246:9001/cgi-bin/admin?admin=admin&msg=coins&coins=600&uId=11929516
    充积分
    http://192.168.43.246:9001/cgi-bin/admin?admin=admin&msg=score&score=100000&uId=11929516
    充钻石
    http://192.168.43.246:9001/cgi-bin/admin?admin=admin&msg=diamond&diamond=100&uId=11482784
    充师徒积分
    http://192.168.43.246:9001/cgi-bin/admin?admin=admin&msg=MasterScore&score=10000&uId=11482784
    绑定手机号
    http://192.168.43.246:9001/cgi-bin/admin?admin=admin&msg=setPhone&phone=18201143169&uId=11929516
