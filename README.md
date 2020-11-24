#### 数据权限模块主要接口：
##### 1.子应用启动权限分配

接口地址：/subApp/update

描述：指定这个子应用可以由哪些身份的人发起，这个身份可以是用户id、部门id、角色id。

入参：
```
public class SubAppVo {
    //要分配的子应用实体
    private FnEntSubApp subApp;
    //要分配给这些用户。（不给任何用户/部门/角色/服务分配时前端应传空列表，而不是null）
    private List<String> userIds;
    //要分配给这些部门
    private List<String> deptIds;
    //要分配给这些角色
    private List<String> roles;
    //要分配给这些服务。如：任务服务
    private List<String> service;
    //权限类型：1 手动开始的权限；2 自动
    private String type;
}
```
大致流程：

1. 如果选了手动开始模式，且为给任何实体分配，则该子应用置为所有人可发起。
2. 然后分别向subAppUser、subAppDept、subAppRole插入分配关系记录，插入方式为先根据子应用id清空此subApp的所有分配记录，再把本次分配的记录全部插入。

##### 2.用户登录泛能App后查看自己有哪些可进入的主应用
接口地址：/appUser/getAppByUserId

描述：权限分配时并不是按照主应用分配而是子应用，所以需要判断该用户是否可以发起主应用下面某个子应用，如果可以这个主应用就可以进入，应该返给前端。

入参：

```
public class QueryAppVo {

    //用户所属的部门（不能传null）
    private List<String> deptIds;

    //用户的角色
    private List<String> roleIds;

    //用户id
    private String userId;

    //应用id（查询子应用时传，这里无用）
    private String appId;
}

```
大致流程：

1. 按用户的多个身份查询他可以发起哪些子应用，并把sub_app表和app表连接查询后返回app表的主应用。
2. 由于按照多个身份查到的可进入主应用可能重复，最后需要根据主键id去重后返给前端。

##### 3.用户从泛能App点击主应用图标后查看有哪些子应用
接口地址：/subAppUser/getUsersBySubAppId

描述：与上面类似，也是从部门/角色/用户id查出此用户可发起哪些子应用，根据subAppId去重后直接返回，无需与主应用表连接返回主应用。

##### 4.配置端点击数据权限分配时弹出组织架构及哪些人已经有权限
1. 查询哪些部门可有此子应用的发起权限：

    接口地址：/fnentsubappdept/getDeptsBySubAppId
    
2. 查询哪些角色可有此子应用的发起权限：

    接口地址：/subAppRole/getRolesBySubAppId
3. 查询哪些用户可有此子应用的发起权限：

    接口地址：/subAppUser/getUsersBySubAppId

以上入参都是：subAppId

#### cim模块主要接口：
##### 1.设备单选
接口地址：/cim/getDevices

入参：请求头的entId字段

描述：从cim服务查询此企业的所有设备和所有设备类型，去掉cim中的虚拟设备类型后，再经过处理后将设备按照类型分到不同的集合。支持前端的二级选择，即先选择类型，再选择该类型下的具体设备。

返回结果：List<DeviceSelectVo>。
```
public class DeviceSelectVo {
    //设备类型名称
    private String typeName;
    //该类型所有设备的编码
    private List<String> deviceIds;
}
```
#### 导入上次数据模块：
##### 1.用户点击提交表单数据后另存一份数据到history_page_data表
接口地址：/busformmanage/saveFormData

描述：bpmn服务调用saveFormData接口，在这个接口里又会再调用HistoryPageDataService的insertOrUpdate方法。
```
//存一下本次的数据，为以后导入历史数据用
historyPageDataService.insertOrUpdate(vo.getPageId(),vo.getCreateBy(),vo.getUpdateData(),vo.getVersion());
```
传入参数：表单pageId，表单提交人userId，表单内容pgaeJson，表单版本version。（如果已有记录和pageId、userId、version一样，则更新pageJson，否则插入新记录）

##### 2.用户点击导入上次数据按钮后返回上次其填写的pageJson
接口地址：/busformmanage/saveFormData

入参：表单pageId，表单提交人userId，表单版本version。

描述：根据入参获取pageJson，如果返回为null则表示无上次提交记录，提示用户。（外链下填写没有userId，应当禁掉导入上次数据功能）。

#### 用户列表模块（user_proc表）：
##### 1. 用户根据类别查询与自己相关的流程记录
接口地址：/userProc/page

传入参数结构：
```
public class QueryPage<T> {
    //无用
    private Integer draw;
    //每页条数
    private Integer pageSize;
    //查第几页
    private Integer pageNumber;
    //查询条件
    private UserAndState queryEntity;
}

public class UserAndState {
    //发起查询的用户id
    private String userId;
    //用户查询哪个tab页下的流程记录。1-待办，2-我发起的，3-我处理的，4-抄送我的
    private Integer state;
    //结果按创建时间倒序还是正序返回。0-倒序，1-升序
    private Integer sortRule;
}

```
描述：除了返回processId、taskId等这些查看详情的必须参数外，还需要在程序中根据不同用户源查询每条流程记录的发起人、状态拼接文案。其中状态有：已完成、已拒绝、已完成，如果是工单应用的流程记录就还有"已过期"（流程没结束，但执行时间超过工单有效期）。

##### 2. bpmn服务在流程结束时设置相关记录状态为已完成
接口地址：/bussiness/setFinishTrue

入参：procInsId

描述：根据流程id将记录finish字段设置为true。

##### 3. 用户处理完待办后bpmn将流程记录从"我的待办"改为其他状态
接口地址：/bussiness/setStatus

入参：procInsId、userId、taskId、state

描述：根据procInsId、userId、taskId找到记录，并将这条记录的类别设置为state。

##### 4. 工单流程超时后将记录设置为过期状态
接口地址：/bussiness/setExpireByInstId

入参：instId、taskId

描述：根据流程实例的instId找到待办类记录将isExpire字段设为true，并将这个taskId的记录的state改为3移到"已处理的"tab下。

##### 4. 流程任务完成一个后，bpmn服务给下一任务负责人插入待办记录
接口地址：/bussiness/doitem

入参如下：
```
UserProc proc      //user_proc表对应的实体类，userId字段实际是逗号隔开的多个用户id
String createUser  //此流程的发起人id
String taskNodeId  //当前任务节点id，用于获取当前任务的表单名称
String bpmnModelId //流程模型id
```
描述：拆分多个用户id为单个，并对每个用户构造UserProc对象插入user_proc表。再根据用户源调用消息模块发送对应的工作流消息和友盟消息。

#### 消息通知模块
略

#### 表单复制
接口地址：/entFormPage/copyFormToTarget

入参如下:
```
public class CopyFormDTO {
    //生成的新表单的名称,前端拼好后传入
    String pageName;
    //源表单的id
    String pageId;
    //目标子应用id，即要将表单拷贝到哪个子应用下
    String targetSubAppId;
}
```
描述：（1）首先进行新表单名字长度、是否和目标子应用下表单重名等判断（2）然后通过redis锁机制判断是否已经有人正在操作（3）如果获取锁成功则根据源表单的pageJson生成新表单插入到目标子应用下（4）最后调用表单属性处理方法维护新表单的属性、控件属性等以及生成对应的数据库表。
