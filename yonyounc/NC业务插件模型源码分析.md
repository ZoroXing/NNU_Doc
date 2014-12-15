##NC业务插件模型源码分析

###1. Summary
为了增加NC程序事件响应的可扩展性，在Java事件响应的前后添加一些必要的操作。

###2. NC事件监听器数据库设计

![数据库表设计](https://github.com/ZoroXing/NNU_Doc/blob/master/picture/yonyounc/eventlistener.png)

表中字段的含义如下表：

**pub_eventlistener**

  字段 | 含义 | 举例
 ------|------|------
 pk_eventlistener  | 主键 | 1001ZF1000000000099E
 name              | 名称 | 同步经销商云平台商家
 implcalssname     | 实现类的全限定名，需实现IBusinessListener接口 | OrgInsertAfterListener
 pk_eventtype      | 外键，连接pub_eventtype表 | 1018Z01000000000LLB9
 owner             | 外键，连接dap_dapsystem   | EC99
 
 **pub_eventtype**
 
  字段 | 含义 | 举例
 ------|------|------
 pk_eventtype  | 主键 | 1018Z01000000000LLB9
 eventtypename | 事件类型名称 | 新增后
 sourceid      | 元数据ID     | 945f38b6-48ec-43e6-bb09-77ec89a3728f
 owner         |  ?            | 1010
 
 **dap_dapsystem**
 
   字段 | 含义 | 举例
 ------|------|------
 moduleid | 主键，与pub_eventlistener表的owner关联 | EC99
 devmodule | 模块名称 | eschain
 
###3 基本代码分析

####3.1 代码配置组合

为了说明**业务插件**代码的运行原理，我们以品牌(bd_branddoc)插入后业务插件为例进行说明。VC6.0中大部分单据界面都需要一个Spring.xml 文件来配置界面，对界面进行布局。这个xml文件必须放在client文件夹下，路径和注册功能节点是注册的“BeanConfigPath”参数一致。

![应用管理平台->开发配置工具->功能注册](https://github.com/ZoroXing/NNU_Doc/blob/master/picture/yonyounc/brandfunc.png)

在**uiuapbd_brand.jar**文件中找到对应的spring配置文件：branddoc_globe.xml,主要配置如下：

```
<beans>
	<!--
		界面布局总装###########################################################
	-->
	<bean id="container" class="nc.ui.uif2.TangramContainer"
		init-method="initUI">
		<property name="tangramLayoutRoot">
			<bean class="nc.ui.uif2.tangramlayout.node.CNode">
				<property name="component" ref="brandlist"></property>
			</bean>
		</property>
		<property name="editActions">
			<list>
				<ref bean="saveAction" />
				......
			</list>
		</property>
		<property name="model" ref="batchBillTableModel"></property>
	</bean>

	<bean id="saveAction" class="nc.ui.uif2.actions.batch.BatchSaveAction">
		<property name="model" ref="batchBillTableModel" />
		<property name="editor" ref="brandlist" />
		<property name="validationService" ref="validatorServicer"></property>
	</bean>
	
	<!-- 批量操作应用模型 -->
	<bean id="batchBillTableModel" class="nc.ui.uif2.model.BatchBillTableModel">
		<property name="service" ref="batchModelService"></property>
		<property name="businessObjectAdapterFactory" ref="objectadapterfactory"></property>
		<property name="context" ref="context"></property>
	</bean>
	
	<!-- 应用服务类，负责进行模型操作的处理 -->
	<bean id="batchModelService" class="nc.ui.bd.material.branddoc.model.BrandDocModelServicer" />
	......
</beans>
```

####3.2 代码调用过程

当点击保存按钮时，会调用nc.ui.uif2.actions.batch.BatchSaveAction类(父类为：javax.swing.Action)的doAction方法。基本的类调用关系如下图：

![品牌业务插件调用过程](https://github.com/ZoroXing/NNU_Doc/blob/master/picture/yonyounc/brandplugincls.png)


```
nc.ui.uif2.actions.batch.BatchSaveAction#doAction()
↓
nc.ui.uif2.model.BatchBillTableModel#save()
↓
nc.ui.bd.material.branddoc.model.BrandDocModelServicer#batchSave
↓
nc.impl.bd.material.branddoc.BrandDocManagerImpl#batchSaveBrandDoc
↓
nc.impl.bd.material.branddoc.BrandDocManagerImpl(父类：nc.bs.bd.baseservice.md.BatchBaseService)#batchSave
↓
insertVO
↓
fireBeforeInsertEvent
fireAfterInsertEvent
↓
nc.bs.businessevent.bd.BDCommonEventUtil#dispatchInsertBeforeEvent
nc.bs.businessevent.bd.BDCommonEventUtil#dispatchInsertAfterEvent
↓
nc.bs.businessevent.EventDispatcher#fireEvent() // 静态方法
```

####3.4 EventDispatcher事件派发过程

业务插件的事件派发过程主要有nc.bs.businessevent.EventDispatcher类的静态方法fireEvent完成，核心代码如下：

```
public static void fireEvent(IBusinessEvent event) throws BusinessException {

		writeDebugLog(event);
		// 从数据库中获取event listener 数据映射关系
		Map<String, UnionVO> classNameMap = getListenersInfo(event.getSourceID(), event.getEventType());
		if (classNameMap == null || classNameMap.isEmpty())
			return;

		String className = null, devModuleCode = null;
		try {
			String currgroup_localtype = getCurrGroupLocalType();
			String currgroup_industrytype = getCurrGroupIndustryType();
			
			for (Iterator<String> iter = classNameMap.keySet().iterator(); iter
					.hasNext();) {
			  // 获取event listener 的实现类名称
				className = iter.next();
				UnionVO unionvo = classNameMap.get(className);
				devModuleCode = unionvo.getDevmodulecode();
				String classInfoMsg = "Plugin class [" + className + "] modulecode [" + devModuleCode + "] ";
				/*
				 * 执行优先级为1/2/3/4的插件并集： 
				 * 行业为水平+本地化为国际化
				 * 行业为水平+ 本地化（如果本地化能匹配）
				 * 匹配行业（如果行业能匹配）+ 本地化为国际化 
				 * 匹配行业（如果行业匹配） + 本地化（如果本地化能匹配）
				 * 
				 * 插件中，行业为空或者为默认值综合控股集团按水平算，本地化为空或为中国按国际化算
				 */
				String localtype = StringUtil.isEmpty(unionvo.getLocaltype()) ? "" : unionvo.getLocaltype();
				String industrytype = StringUtil.isEmpty(unionvo.getIndustrytype()) ? "" : unionvo.getIndustrytype();
				int level = LocalAndIndustryLevelQueryUtil.getLevelByLocalAndIndustry(
						localtype, industrytype, currgroup_localtype, currgroup_industrytype);
				if (level == 1 || level == 2 || level == 3 || level == 4) {
					Logger.debug(classInfoMsg + " begin.");
					Long start = System.currentTimeMillis();
					// 利用反射获取IBusinessListener实例
					IBusinessListener instance = getInstanceByClassName(className, devModuleCode);
				  // 执行Event listener 的doAction方法
					instance.doAction(event);
					Logger.debug(classInfoMsg + " end successfully.cost time : " + (System.currentTimeMillis() - start));
				}
			}
		} catch (BusinessException e) {
			Logger.error(getErrorMsg(event, className), e);
			throw e;
		} catch (RuntimeException e) {
			Logger.error(getErrorMsg(event, className), e);
			throw e;
		}
	}
```

fireEvent方法的主要任务是说先从表：pub_eventlistener，pub_eventtype与dap_dapsystem获取相应的数据并保存到Map<String, UnionVO>中

```
nc.bs.businessevent.EventDispatcher#getListenersInfo() // 返回值：Map<String, UnionVO>
↓
getEventType_ClassNameMap() // 返回值为： Map<String, Map<String, UnionVO>>
↓
getAllListenersMap() 
↓
getAllListenersByOrder()
↓
nc.impl.businessevent.EventListenerQryServiceImpl#getAllListeners()

//getAllListeners的代码实现如下：
public UnionVO[] getAllListeners() throws BusinessException {
		String sql = "select T1.sourceid as sourceid,T1.eventtypecode as eventtype," +
					"T0.implclassname as implclassname,T0.operindex as operindex," +
					"M.devmodule as devmodulecode," +
					"T0.localtype as localtype,T0.industrytype as industrytype " +
				     "from pub_eventlistener T0 " +
				     	"left outer join pub_eventtype T1 on T0.pk_eventtype =T1.pk_eventtype " +
				     	"left join dap_dapsystem  M on M.moduleid=T0.owner "+
				     "where T0.enabled = 'Y' or T0.enabled = 'y'";
		List<UnionVO> vos = (List<UnionVO>) getBaseDAO().executeQuery(sql,
				new BeanListProcessor(UnionVO.class));
		return vos.toArray(new UnionVO[0]);
	}
```

getEventType_ClassNameMap返回Map<String, Map<String, UnionVO>>的映射关系，他们的含义如下：

       | 含义    | 举例
-------|---------|---------
Key    | SourceId+'@'+eventtytpe | 3ee53558-6398-4096-a91f-c7aa00e93701@新增后
Value  | implclassname与UnionVo的映射 | 

其中SourceId的获取是在nc.impl.bd.material.branddoc.BrandDocManagerImpl实例创建时指定的：

```
	public BrandDocManagerImpl() {
		super(IBDMetaDataIDConst.BRANDDOC);
	}
```

 nc.itf.bd.pub.IBDMetaDataIDConst中包含了所有模块对应的SourceId常量值。
 
 
 以上。

