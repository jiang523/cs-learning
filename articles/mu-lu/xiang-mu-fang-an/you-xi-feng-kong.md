# 游戏风控

常见的黑产行为

1. 苹果代充，因为苹果的汇率不是实时的，黑产可以通过外币和苹果的汇率差来做代充
2. 苹果退款，苹果每个账号可以一次退款，退款了但是游戏内的道具还在，黑产通过大量新账号去刷退款，会导致直接的收入损失，另外退款率过高苹果会提高抽成比例，导致游戏的收入遭到巨大损失
3. 用票仓刷苹果票据，苹果票据里有订单透传的信息，黑产试图伪造苹果的票据
4. 对安卓SDK进行破包，伪造请求  ==> 通过设备指纹识别

现有的问题

1. 经常需要查询任意时间窗口内用户的数据，且风控对性能要求高，不能阻塞主流程，因此需要更好的窗口查询数据库----时序数据库(OpenTSDB)
2. 规则引擎如何建立？如何尽快的应对突发的风险事件？规则是否需要进行版本控制？
3. 风险用户如何更好的识别？分值系统如何建立？
4. 对风险用户是如何处置的？封号？人脸识别？
5. 黑名单白名单过滤？
6. 设备指纹是如何关联的？请求数据如何异步上报给数据平台？
