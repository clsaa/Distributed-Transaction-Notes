# 5-TCC型方案

## 5.1 介绍

>TCC方案属于两阶段型/补偿型

### 5.1.1 实现

![image](http://clsaa-distributed-transaction-img-bed-1252032169.cossh.myqcloud.com/TCC1.png)

* 一个完整的业务活动由一个主业务服务与若干从业务服务组成
* 主业务服务负责发起并完成整个业务活动
* 从业务服务提供TCC型业务操作
* 业务活动管理器控制业务活动的一致性,它登记业务活动中的操作,并在业务活动提交时确认所有的TCC型操作的confirm操作,在业务活动取消时调用所有TCC型操作的cancel操作.

### 5.1.2 成本

* 业务活动结束时confirm或cancel操作的执行成本
* 业务活动日志成本

### 5.1.3 适用范围

* 强隔离性,严格一致性要求的业务活动
* 适用于执行时间较短的业务,如处理账户,收费等业务

### 5.1.4 用到的服务模式

* TCC操作
* 幂等操作
* 可补偿操作
* 可查询操作

### 5.1.5 方案特点

* 不与具体的服务框架耦合(RPC架构通用)
* 位于业务服务层,而非资源层
* 可以灵活选择业务资源的锁定粒度
* TCC里对每个服务资源操作的是本地事务,数据被lock的时间短,可扩展性好(可以说是为独立部署的SOA服务而设计的)

### 5.1.6 行业应用案例

* 支付宝XTS(蚂蚁金融云的分布式事务服务DTS)

## 5.2 TCC框架

>[GitHub上一个开源的TCC框架实现](https://github.com/changmingxie/tcc-transaction),以此项目为基础讲解TCC框架的实现(主要是在项目源码上添加一些注释)

## 5.3 TCC应用实例

### 5.3.1 业务流程

>以订单处理流程为例:账户扣款->使用红包优惠券->订单状态改变为例

![image](http://clsaa-distributed-transaction-img-bed-1252032169.cossh.myqcloud.com/TCC3.png)

>一个主服务调用从业务服务,被TCC控制的方法都应该具有三种方法:主方法(try方法)/确认方法(confirm方法)/取消方法(cancel方法)

1. 修改支付记录状态
2. 修改订单状态
3. 远程调用点券服务
4. 远程调用积分服务
5. 若远程调用存在异常

### 5.2.2 使用TCC-TRASACTION示例

```java
@Service("pointAccountService")
public class pointAccountServiceImpl implements pointAccountService{

    private static final Logger LOG = LoggerFactory.getLogger(pointAccountServiceImpl.class);

    @Autowired
    private pointAccountDao pointAccountDao;

    @Autowired
    private pointAccountHistoryDao pointAccountHistoryDao;

    @Override
    public void saveData(pointAccount pointAccount) {
        pointAccountDao.insert(pointAccount);
    }

    @Override
    public void updateData(pointAccount pointAccount) {
        pointAccountDao.update(pointAccount);
    }


    /**
     * 积分账户加款 Trying
     * @param transactionContext
     * @param userNo
     * @param pointAmount
     * @param requestNo
     * @param bankTrxNo
     * @param trxType
     * @param remark
     * @throws BizException
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    @Compensable(confirmMethod = "confirmCreditToPointAccountTcc",cancelMethod = "cancelCreditToPointAccountTcc")
    public void creditToPointAccountTcc(TransactionContext transactionContext, String userNo, Integer pointAmount, String requestNo, String bankTrxNo, String trxType, String remark) throws BizException {

        LOG.info("===>creditToPointAccountTcc TRYING begin");

        //根据商户编号获取商户积分账户
        pointAccount pointAccount = pointAccountDao.getByUserNo(userNo);
        if (pointAccount == null){//如果不存在商户积分账户,创建一条新的积分账户
            pointAccount = new pointAccount();
            pointAccount.setBalance(0);
            pointAccount.setUserNo(userNo);
            pointAccount.setStatus(PublicEnum.YES.name());
            pointAccount.setCreateTime(new Date());
            pointAccount.setId(StringUtil.get32UUID());
            pointAccountDao.insert(pointAccount);
        }

        pointAccountHistory pointAccountHistory = pointAccountHistoryDao.getByRequestNo(requestNo);
        // 幂等判断
        if ( pointAccountHistory == null ){//防止多次提交
            pointAccountHistory = new pointAccountHistory();
            pointAccountHistory.setId(StringUtil.get32UUID());
            pointAccountHistory.setCreateTime(new Date());
            pointAccountHistory.setStatus(PointAccountHistoryStatusEnum.TRYING.name());//消息不可用
            pointAccountHistory.setAmount(pointAmount);///积分账户变动额
            pointAccountHistory.setBalance(pointAccount.getBalance() + pointAmount);
            pointAccountHistory.setBankTrxNo(bankTrxNo);//银行流水号
            pointAccountHistory.setRequestNo(requestNo);//请求号
            pointAccountHistory.setFundDirection(PointAccountFundDirectionEnum.ADD.name());
            pointAccountHistory.setTrxType(trxType);
            pointAccountHistory.setRemark(remark);
            pointAccountHistory.setUserNo(userNo);
            pointAccountHistoryDao.insert(pointAccountHistory);
        }else if (PointAccountHistoryStatusEnum.CANCEL.name().equals(pointAccountHistory.getStatus())){
            //如果是取消的,有可能是之前的业务出现异常问题而取消,那么重试阶段,再将状态更新为TYING状态,而不是重新创建一条
            LOG.info("之前因为业务问题取消后,又重试的{}" , pointAccountHistory.getBankTrxNo());
            pointAccountHistory.setStatus(PointAccountHistoryStatusEnum.TRYING.name());
            this.pointAccountHistoryDao.update(pointAccountHistory);
        }
        //添加一条不可用的积分账户流水
        LOG.info("===>creditToPointAccountTcc TRYING end");
    }

    /**
     * 积分账户增加确认
     * @param transactionContext
     * @param userNo
     * @param pointAmount
     * @param requestNo
     * @param bankTrxNo
     * @param trxType
     * @param remark
     * @return
     * @throws BizException
     */

    @Transactional(rollbackFor = Exception.class)
    public void confirmCreditToPointAccountTcc(TransactionContext transactionContext, String userNo, Integer pointAmount, String requestNo, String bankTrxNo, String trxType, String remark) throws BizException {

        LOG.info("===>confirmCreditToPointAccountTcc begin");
        //根据请求号获取账户基本流水
        pointAccountHistory pointAccountHistory = pointAccountHistoryDao.getByRequestNo(requestNo);
        // 幂等判断
        if ( pointAccountHistory == null  || PointAccountHistoryStatusEnum.CONFORM.name().equals(pointAccountHistory.getStatus())){//该笔交易流水已处理过,不需再处理
            return;
        }

        pointAccountHistory.setStatus(PointAccountHistoryStatusEnum.CONFORM.name());
        pointAccountHistoryDao.update(pointAccountHistory);

        pointAccount pointAccount = pointAccountDao.getByUserNo(userNo);//获取用户积分账户
        pointAccount.setBalance(pointAccount.getBalance() + pointAmount);//增加账户余额
        pointAccountDao.update(pointAccount);

        LOG.info("===>confirmCreditToPointAccountTcc end");

    }
    /**
     *积分账户增加回滚
     * @param transactionContext
     * @param userNo
     * @param pointAmount
     * @param requestNo
     * @param bankTrxNo
     * @param trxType
     * @param remark
     * @throws BizException
     */
    @Transactional(rollbackFor = Exception.class)
    public void cancelCreditToPointAccountTcc(TransactionContext transactionContext, String userNo, Integer pointAmount, String requestNo, String bankTrxNo, String trxType, String remark) throws BizException {
        LOG.info("===>cancelCreditToPointAccountTcc begin");
        pointAccountHistory pointAccountHistory = pointAccountHistoryDao.getByRequestNo(requestNo);
        // 幂等判断
        if ( pointAccountHistory == null  || !PointAccountHistoryStatusEnum.TRYING.name().equals(pointAccountHistory.getStatus())){//该笔交易流水已处理过,不需再处理
            return;
        }

        pointAccountHistory.setStatus(PointAccountHistoryStatusEnum.CANCEL.name());
        pointAccountHistoryDao.update(pointAccountHistory);
        LOG.info("===>cancelCreditToPointAccountTcc end");
    }

    /**
     * 积分账户加款 Trying
     * @param userNo
     * @param pointAmount
     * @param requestNo
     * @param bankTrxNo
     * @param trxType
     * @param remark
     * @throws BizException
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void creditToPointAccount(String userNo, Integer pointAmount, String requestNo, String bankTrxNo, String trxType, String remark) throws BizException {

        //根据商户编号获取商户积分账户
        pointAccount pointAccount = pointAccountDao.getByUserNo(userNo);
        if (pointAccount == null){//如果不存在商户积分账户,创建一条新的积分账户
            pointAccount = new pointAccount();
            pointAccount.setBalance(0);
            pointAccount.setUserNo(userNo);
            pointAccount.setStatus(PublicEnum.YES.name());
            pointAccount.setCreateTime(new Date());
            pointAccount.setId(StringUtil.get32UUID());
            pointAccountDao.insert(pointAccount);
        }


    //添加一条积分历史
    pointAccountHistory pointAccountHistory = pointAccountHistoryDao.getByRequestNo(requestNo);
    if ( pointAccountHistory == null ){//防止多次提交
            pointAccountHistory = new pointAccountHistory();
            pointAccountHistory.setId(StringUtil.get32UUID());
            pointAccountHistory.setCreateTime(new Date());
            pointAccountHistory.setStatus(PointAccountHistoryStatusEnum.CONFORM.name());//可用
            pointAccountHistory.setAmount(pointAmount);///积分账户变动额
            pointAccountHistory.setBalance(pointAccount.getBalance() + pointAmount);
            pointAccountHistory.setBankTrxNo(bankTrxNo);//银行流水号
            pointAccountHistory.setRequestNo(requestNo);//请求号
            pointAccountHistory.setFundDirection(PointAccountFundDirectionEnum.ADD.name());
            pointAccountHistory.setTrxType(trxType);
            pointAccountHistory.setRemark(remark);
            pointAccountHistory.setUserNo(userNo);
            pointAccountHistoryDao.insert(pointAccountHistory);
        }

        //增加积分账户
        pointAccount.setBalance(pointAccount.getBalance() + pointAmount);//增加账户余额
        pointAccountDao.update(pointAccount);
    }

    @Override
    public pointAccount getDataById(String id) {
        return pointAccountDao.getById(id);
    }

    @Override
    public PageBean listPage(PageParam pageParam, pointAccount pointAccount) {
        Map<String, Object> paramMap = new HashMap<String, Object>();
        return pointAccountDao.listPage(pageParam, paramMap);
    }
}
```