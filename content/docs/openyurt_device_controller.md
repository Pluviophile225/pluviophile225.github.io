## 2021/2/3

### Crd

#### device

##### DeviceSpec

```go
// DeviceSpec defines the desired state of Device
// 定义了Device的希望状态
type DeviceSpec struct {
	Description string `json:"description,omitempty"`
	// Admin state (locked/unlocked)
    // Admin状态（lock/unlocked）
	AdminState AdminState `json:"adminState,omitempty"`
	// Operating state (enabled/disabled)
    // 运行状态(enabled/disabled)
	OperatingState OperatingState `json:"operatingState,omitempty"`
	// A map of supported protocols for the given device
    // 给定设备支持的协议映射
	Protocols map[string]ProtocolProperties `json:"protocols,omitempty"`
	// Other labels applied to the device to help with searching
    // 应用于设备以帮助搜索的其他标签
	Labels []string `json:"labels,omitempty"`
	// Device service specific location (interface{} is an empty interface so
	// it can be anything)
    // 设备服务特定位置（接口{}是一个空接口，因此它可以是任何东西）
	Location string `json:"location,omitempty"`
	// Associated Device Service - One per device
    // 设备相关服务-每个设备一个
	Service string `json:"service"`
	// Associated Device Profile - Describes the device
    // 关联的设备配置文件-描述这个设备
	Profile string `json:"profile"`
	// TODO support the following field
	// A list of auto-generated events coming from the device
	// AutoEvents     []AutoEvent                   `json:"autoEvents"`
    // 设备属性
	DeviceProperties map[string]DesiredPropertyState `json:"deviceProperties,omitempty"`
}
```

##### DeviceStatus

```go
// DeviceStatus defines the observed state of Device
// DeviceStatus 定义设备的观察状态
type DeviceStatus struct {
	// Time (milliseconds) that the device last provided any feedback or
	// responded to any request
    // 设备上次提供任何反馈或回应任何请求的时间（毫秒）
	LastConnected int64 `json:"lastConnected,omitempty"`
	// Time (milliseconds) that the device reported data to the core
	// microservice
    // 	设备向核心微服务报告数据的时间（毫秒）
	LastReported int64 `json:"lastReported,omitempty"`
	// AddedToEdgeX indicates whether the object has been successfully
	// created on EdgeX Foundry
    // AddedToEdgeX 指示对象是否已成功在 EdgeX Foundry 上创建
	AddedToEdgeX     bool                           `json:"addedToEdgeX,omitempty"`
	DeviceProperties map[string]ActualPropertyState `json:"deviceProperties,omitempty"`
	Id               string                         `json:"id,omitempty"`
}
```

##### DesiredPropertyState

```go
type DesiredPropertyState struct {
    // Device属性的名称
	Name         string `json:"name"`
    // 修改时候要用Put请求，这个是Put请求的URL地址
	PutURL       string `json:"putURL,omitempty"`
    // 这个是属性对应的值
	DesiredValue string `json:"desiredValue"`
}
```

##### ActualPropertyState

```go
type ActualPropertyState struct {
    // Device属性的名称
	Name        string `json:"name"`
    // 查询时候要用Get请求，这个是Get请求的URL地址
	GetURL      string `json:"getURL,omitempty"`
    // 属性对应的值
	ActualValue string `json:"actualValue"`
}
```

#### deviceprofile

##### DeviceResource

```go
type DeviceResource struct {
    // 资源描述
	Description string            `json:"description"`
    // 资源名称
	Name        string            `json:"name"`
    // 资源标签
	Tag         string            `json:"tag,omitempty"`
    // 资源属性配置
	Properties  ProfileProperty   `json:"properties"`
	Attributes  map[string]string `json:"attributes,omitempty"`
}
```

##### ProfileProperty

```go
type ProfileProperty struct {
    // 属性值
	Value PropertyValue `json:"value"`
    // 
	Units Units         `json:"units,omitempty"`
}
```

##### PropertyValue

```go
type PropertyValue struct {
    // ValueDescriptor Type of property after transformations
    // 值描述符 转换后属性的类型
	Type         string `json:"type,omitempty"`  
    // Read/Write Permissions set for this property
    // 为此属性设置的读/写权限
	ReadWrite    string `json:"readWrite,omitempty"`   
    // Minimum value that can be get/set from this property
    // 可以从此属性获取/设置的最小值
	Minimum      string `json:"minimum,omitempty"` 
    // Maximum value that can be get/set from this property
    // 可以从此属性获取/设置的最大值
	Maximum      string `json:"maximum,omitempty"`   
    // Default value set to this property if no argument is passed
    // 如果未传递参数，则默认值设置为此属性
	DefaultValue string `json:"defaultValue,omitempty"` 
    // Size of this property in its type  
    // (i.e. bytes for numeric types, characters for string types)
    // 此属性在其类型中的大小 （即数字类型的字节，字符串类型的字符）
	Size         string `json:"size,omitempty"`  
    // Mask to be applied prior to get/set of property
    // 在获取/属性集之前应用的掩码
	Mask         string `json:"mask,omitempty"`      
    // Shift to be applied after masking, prior to get/set of property
    // 在设置掩码后，在获取/设置属性之前应用的移位
	Shift        string `json:"shift,omitempty"`
    // Multiplicative factor to be applied after shifting, prior to get/set of property
    // 在移位后，在获取/属性集之前应用的乘法因子
	Scale        string `json:"scale,omitempty"`      
    // Additive factor to be applied after multiplying, prior to get/set of property
    // 乘法后，在获得/属性集之前应用的加性因子
	Offset       string `json:"offset,omitempty"`  
    // Base for property to be applied to, leave 0 for no power operation
    // (i.e. base ^ property: 2 ^ 10)
    // 要应用的属性的基础，保留 0 表示无电源操作 （即基数^属性：2^10）
	Base         string `json:"base,omitempty"`         
	// Required value of the property, set for checking error state. Failing an
	// assertion condition wil  l mark the device with an error state
    // 属性的必需值，为检查错误状态而设置。失败断言条件将设备标记为错误状态
	Assertion     string `json:"assertion,omitempty"`
	Precision     string `json:"precision,omitempty"`
    // FloatEncoding indicates the representation of floating value of reading. 
    // It should be 'Base64'   or 'eNotation'
    // 浮点编码表示读取的浮点值的表示形式。它应该是“Base64”或“电子符号”
	FloatEncoding string `json:"floatEncoding,omitempty"` 
	MediaType     string `json:"mediaType,omitempty"`
}
```

##### Units

```go
type Units struct {
	Type         string `json:"type,omitempty"`
	ReadWrite    string `json:"readWrite,omitempty"`
	DefaultValue string `json:"defaultValue,omitempty"`
}
```

##### Command

```go
type Command struct {
	// EdgeXId is a unique identifier used by EdgeX Foundry, such as a UUID
    // EdgeXId 是 EdgeX Foundry 使用的唯一标识符，例如 UUID
	EdgeXId string `json:"id,omitempty"`
	// Command name (unique on the profile)
    // 命令名称（在配置文件中唯一）
	Name string `json:"name,omitempty"`
	// Get Command
    // Get指令
	Get Get `json:"get,omitempty"`
	// Put Command
    // Put指令
	Put Put `json:"put,omitempty"`
}
```

##### Action

```go
type Action struct {
	// Path used by service for action on a device or sensor
    // 服务用于对设备或传感器执行操作的路径
	Path string `json:"path,omitempty"`
	// Responses from get or put requests to service
    // 从Get或PUt请求到服务的响应
	Responses []Response `json:"responses,omitempty"`
	// Url for requests from command service
    // 来自命令服务的请求的 URL
	URL string `json:"url,omitempty"`
}
```

##### Put

```go
type Put struct {
    // Action用来定义Put和Get请求
	Action         `json:",inline"`
    // 请求的参数数组
	ParameterNames []string `json:"parameterNames,omitempty"`
}
```

##### Get

```go
type Get struct {
    // Action用来定义Put和Get请求
	Action `json:",omitempty"`
}
```

##### Response

```go
// Response for a Get or Put request to a service
// Get或Put请求的响应
type Response struct {
    // 响应代码
	Code           string   `json:"code,omitempty"`
    // 响应的描述
	Description    string   `json:"description,omitempty"`
    // 响应的返回列表
	ExpectedValues []string `json:"expectedValues,omitempty"`
}
```

##### ResourceOperation

```go
type ResourceOperation struct {
	Index     string `json:"index,omitempty"`
	Operation string `json:"operation,omitempty"`
	// Deprecated
	Object string `json:"object,omitempty"`
	// The replacement of Object field
	DeviceResource string `json:"deviceResource,omitempty"`
	Parameter      string `json:"parameter,omitempty"`
	// Deprecated
	Resource string `json:"resource,omitempty"`
	// The replacement of Resource field
	DeviceCommand string            `json:"deviceCommand,omitempty"`
	Secondary     []string          `json:"secondary,omitempty"`
	Mappings      map[string]string `json:"mappings,omitempty"`
}
```

##### ProfileResource

```go
type ProfileResource struct {
	Name string              `json:"name,omitempty"`
	Get  []ResourceOperation `json:"get,omitempty"`
	Set  []ResourceOperation `json:"set,omitempty"`
}
```

##### DeviceProfileSpec

```go
type DeviceProfileSpec struct {
    // 这个是对设备配置的描述
	Description string `json:"description,omitempty"`
	// Manufacturer of the device
    // 设备制造商
	Manufacturer string `json:"manufacturer,omitempty"`
	// Model of the device
    // 设备的模型
	Model string `json:"model,omitempty"`
	// EdgeXLabels used to search for groups of profiles on EdgeX Foundry
    // 设备资源列表
	EdgeXLabels     []string         `json:"labels,omitempty"`
	DeviceResources []DeviceResource `json:"deviceResources,omitempty"`

	// TODO support the following field
    // 配置资源
	DeviceCommands []ProfileResource `json:"deviceCommands,omitempty"`
    // 命令行
	CoreCommands   []Command         `json:"coreCommands,omitempty"`
}
```

##### DeviceProfileStatus

```go
type DeviceProfileStatus struct {
    // 对应EdgeXId
	EdgeXId      string `json:"id,omitempty"`
    // 是否已经加入到EdgeX中
	AddedToEdgeX bool   `json:"addedToEdgeX,omitempty"`
}
```

#### deviceservice

##### Addressable

```go
type Addressable struct {
	// ID is a unique identifier for the Addressable, such as a UUID
    // ID是Addressable的唯一标识，比如UUID
	Id string `json:"id,omitempty"`
	// Name is a unique name given to the Addressable
    // Name是给予Addressable唯一名称
	Name string `json:"name,omitempty"`
	// Protocol for the address (HTTP/TCP)
    // 地址的协议(HTTP/TCP)
	Protocol string `json:"protocol,omitempty"`
	// Method for connecting (i.e. POST)
    // 连接的方式(比如 POST)
	HTTPMethod string `json:"method,omitempty"`
	// Address of the addressable
    // addressable的地址
	Address string `json:"address,omitempty"`
	// Port for the address
    // 地址的端口
	Port int `json:"port,omitempty"`
	// Path for callbacks
    // 回调的路径
	Path string `json:"path,omitempty"`
	// For message bus protocols
    // 提供给message bus协议
	Publisher string `json:"publisher,omitempty"`
	// User id for authentication
    // 用于设备身份验证的用户 ID
	User string `json:"user,omitempty"`
	// Password of the user for authentication for the addressable
    // 用于对可寻址对象进行身份验证的用户密码
	Password string `json:"password,omitempty"`
	// Topic for message bus addressables
    // 消息总线可寻址对象主题
	Topic string `json:"topic,omitempty"`
}
```

##### DeviceServiceSpec

```go
type DeviceServiceSpec struct {
    // Spec的描述内容
	Description string `json:"description,omitempty"`
	// the Id assigned by the EdgeX foundry
	// TODO store this field in the status
    // 由 EdgeX 代工厂分配的 ID
	Id string `json:"id,omitempty"`
	// TODO store this field in the status
    // 设备上次将数据连接到核心的时间（以毫秒为单位）
	LastConnected int64 `json:"lastConnected,omitempty"`
	// time in milliseconds that the device last reported data to the core
    // 设备上次向核心报告数据的时间（以毫秒为单位）
	// TODO store this field in the status
	LastReported int64 `json:"lastReported,omitempty"`
	// operational state - either enabled or disabled
    // 操作状态 - 启用或禁用
	OperatingState OperatingState `json:"operatingState,omitempty"`
	// tags or other labels applied to the device service for search or other
	// identification needs on the EdgeX Foundry
    // 标记或其他标签应用于设备服务以进行搜索或其他EdgeX Foundry的识别需求
	Labels []string `json:"labels,omitempty"`
	// address (MQTT topic, HTTP address, serial bus, etc.) for reaching
	// the service
    // 地址（MQTT topic、HTTP 地址、串行总线等）用于到达服务内容
	Addressable Addressable `json:"addressable,omitempty"`
	// Device Service Admin State
    // 设备服务管理员状态(locked、unlocked)
	AdminState AdminState `json:"adminState,omitempty"`
}
```

##### DeviceServiceStatus

```go
type DeviceServiceStatus struct {
    // 是否添加到EdgeX中
	AddedToEdgeX bool `json:"addedToEdgeX,omitempty"`
}
```

#### valueDescriptor

##### ValueDescriptorSpec

```go
type ValueDescriptorSpec struct {
	Id            string   `json:"id,omitempty"`
	Created       int64    `json:"created,omitempty"`
	Description   string   `json:"description,omitempty"`
	Modified      int64    `json:"modified,omitempty"`
	Origin        int64    `json:"origin,omitempty"`
	Min           string   `json:"min,omitempty"`
	Max           string   `json:"max,omitempty"`
	DefaultValue  string   `json:"defaultValue,omitempty"`
	Type          string   `json:"type,omitempty"`
	UomLabel      string   `json:"uomLabel,omitempty"`
	Formatting    string   `json:"formatting,omitempty"`
	Labels        []string `json:"labels,omitempty"`
	MediaType     string   `json:"mediaType,omitempty"`
	FloatEncoding string   `json:"floatEncoding,omitempty"`
}
```

##### ValueDescriptorStatus

```go
type ValueDescriptorStatus struct {
	// AddedToEdgeX indicates whether the object has been successfully
	// created on EdgeX Foundry
	AddedToEdgeX bool `json:"addedToEdgeX,omitempty"`
}
```

### Controller

#### device_controller

```go
func (r *DeviceReconciler) Reconcile(
	ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := r.Log.WithValues("device", req.NamespacedName)
	var d devicev1alpha1.Device
    // 首先获取这个Device
	if err := r.Get(ctx, req.NamespacedName, &d); err != nil {
		return ctrl.Result{}, err
	}
	log.Info("Reconciling the Device object", "Device", d.GetName(), "AddedToEdgeX", d.Status.AddedToEdgeX)
	// 如果设备状态中AddedToEdgeX为true时
	if d.Status.AddedToEdgeX == true {
		// the device has been added to the EdgeX foundry,
		// check if each device property are in the desired state
        // 这个设备已经被添加到了EdgeX foundry中，检查是否设备中属性是想要的状态
		for _, dps := range d.Spec.DeviceProperties {
			log.Info("getting the actual property state", "property", dps.Name)
            // 根据dps(device的一个属性)来获取这个属性的真实值
			aps, err := getActualPropertyState(dps.Name, &d, r.CoreCommandClient)
			if err != nil {
				return ctrl.Result{}, err
			}
			log.Info("got the actual property state",
				"property name", aps.Name,
				"property getURL", aps.GetURL,
				"property actual value", aps.ActualValue)
			if d.Status.DeviceProperties == nil {
				d.Status.DeviceProperties = map[string]devicev1alpha1.ActualPropertyState{}
			}
            // 先将这个这个值更新到Status中
			d.Status.DeviceProperties[aps.Name] = aps
            // 看看真实值和需要的属性是否相同
			if dps.DesiredValue != aps.ActualValue {
                // 如果真实状态与需要的状态不同
				log.Info("the desired value and the actual value are different",
					"desired value", dps.DesiredValue,
					"actual value", aps.ActualValue)
                // 如果需要状态的PutURL为空的话，PutURL的作用是什么呢？
                // 从CoreCommandClient中拿到PutURL地址
				if dps.PutURL == "" {
					putURL, err := getPutURL(d.GetName(), dps.Name, r.CoreCommandClient)
					if err != nil {
						return ctrl.Result{}, err
					}
					dps.PutURL = putURL
					log.Info("get the desired property putURL",
						"property", dps.Name, "putURL", putURL)
				}
				// set the device property to desired state
                // 向这个URL地址发送Put请求，来将devcie的属性修改为想要的状态
				log.Info("setting the property to desired value", "property", dps.Name)
                // 这里应该是在发HTTP请求吧
				rep, err := resty.New().R().
					SetHeader("Content-Type", "application/json").
					SetBody([]byte(fmt.Sprintf(`{"%s": "%s"}`, dps.Name, dps.DesiredValue))).
					Put(dps.PutURL)
				if err != nil {
					return ctrl.Result{}, err
				}
                // 如果返回的状态是StatusOK的话
				if rep.StatusCode() == http.StatusOK {
					log.Info("successfully set the property to desired value", "property", dps.Name)
					log.Info("setting the actual property value to desired value", "property", dps.Name)
					// if the device property has been successfully set, we will
					// update the Device.Status.DeviceProperties[name] as well
                    // 如果设备属性被成功设置，需要更新Status中的DeviceProperties[name]
					if d.Status.DeviceProperties == nil {
						d.Status.DeviceProperties = map[string]devicev1alpha1.ActualPropertyState{}
					}
					oldAps, exist := d.Status.DeviceProperties[dps.Name]
					if !exist {
						d.Status.DeviceProperties[dps.Name] = devicev1alpha1.ActualPropertyState{
							Name:        dps.Name,
							ActualValue: dps.DesiredValue,
						}
						continue
					}
					oldAps.ActualValue = dps.DesiredValue
					d.Status.DeviceProperties[dps.Name] = oldAps
					log.Info("set the actual property value to desired value", "property", dps.Name)
				}
			}
		}
		return ctrl.Result{}, r.Status().Update(ctx, &d)
	}
	// 如果AddedToEdgeX的状态是false的话
    // 先查看这个设备是否已经存在EdgeX上
	log.Info("Checking if device already exist on the EdgeX", "device", d.GetName())
	_, err := r.GetDeviceByName(d.GetName())
	if err == nil {
        // 这个有个问题，如果已经存在的话，难道不用修改AddedToEdgeX的值嘛？
		log.Info("Device already exists on EdgeX")
		return ctrl.Result{}, nil
	}
	if !clis.IsNotFoundErr(err) {
		log.Info("fail to visit the EdgeX core-metadata-service")
		return ctrl.Result{}, nil
	}
	// 如果在EdgeX上没有找到的话，就尝试加到EdgeX中去
	log.Info("Adding device to the EdgeX", "device", d.GetName())
	edgeXId, err := r.AddDevice(toEdgeXDevice(d))
	if err != nil {
		return ctrl.Result{}, fmt.Errorf("Fail to add Device to EdgeX: %v", err)
	}
	log.Info("Successfully add Device to EdgeX",
		"Device", d.GetName(), "EdgeXId", edgeXId)
    // 更新Id和状态
	d.Status.Id = edgeXId
	d.Status.AddedToEdgeX = true
	return ctrl.Result{Requeue: true}, r.Status().Update(ctx, &d)
}
```

#### deviceprofile_controller

```go
func (r *DeviceProfileReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := r.Log.WithValues("deviceprofile", req.NamespacedName)
	var dp devicev1alpha1.DeviceProfile
    // 在k8s资源中找这个DeviceProfile是否存在
	if err := r.Get(ctx, req.NamespacedName, &dp); err != nil {
		return ctrl.Result{}, err
	}
	// 到EdgeX中找这个设备是否存在
	_, err := r.GetDeviceProfileByName(dp.GetName())
    // 已经存在了的话
	if err == nil {
		log.Info(
			"DeviceProfile already exists on EdgeX")
		return ctrl.Result{}, nil
	}
	
	if !clis.IsNotFoundErr(err) {
		log.Info("Fail to visit the EdgeX core-metadata-service")
		return ctrl.Result{}, nil
	}
	// 添加这个DeviceProfile到EdgeX中
	edgeXId, err := r.AddDeviceProfile(toEdgeXDeviceProfile(dp))
	if err != nil {
		return ctrl.Result{}, fmt.Errorf("Fail to add DeviceProfile to Edgex: %v", err)
	}
	log.V(4).Info("Successfully add DeviceProfile to EdgeX",
		"DeviceProfile", dp.GetName(), "EdgeXId", edgeXId)
    // 修改相关状态
	dp.Spec.Id = edgeXId
	dp.Status.AddedToEdgeX = true
	return ctrl.Result{}, r.Update(ctx, &dp)
}
```

#### deviceservice_controller

```go
func (r *DeviceServiceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := r.Log.WithValues("deviceservice", req.NamespacedName)
	var ds devicev1alpha1.DeviceService
    // 首先在kubernetes环境中找deviceservice
	if err := r.Get(ctx, req.NamespacedName, &ds); err != nil {
		return ctrl.Result{}, err
	}
	// 再去EdgeX中找这个deviceservice
    // 如果找到了
	_, err := r.GetDeviceServiceByName(ds.GetName())
	if err == nil {
		log.Info(
			"DeviceService already exists on EdgeX")
		return ctrl.Result{}, nil
	}
	if !clis.IsNotFoundErr(err) {
		log.Error(err, "fail to visit the EdgeX core-metatdata-service")
		return ctrl.Result{}, nil
	}
	// 如果没找到，就创建一个addressable
	// 1. create the addressable
	add := toEdgeXAddressable(ds.Spec.Addressable)
	_, err = r.GetAddressableByName(add.Name)
	if err == nil {
		log.Info(
			"Addressable already exists on EdgeX")
		return ctrl.Result{}, nil
	}
    // 添加这个addressable
	addrEdgeXId, err := r.AddAddressable(add)
	if err != nil {
		return ctrl.Result{}, fmt.Errorf("Fail to add addressable to EdgeX: %v", err)
	}
	log.V(4).Info("Successfully add the Addressable to EdgeX",
		"Addressable", add.Name, "EdgeXId", addrEdgeXId)
	ds.Spec.Addressable.Id = addrEdgeXId

	// 2. create the DeviceService
    // 再添加这个DeviceService
	dsEdgeXId, err := r.AddDeviceService(toEdgexDeviceService(ds))
	if err != nil {
		return ctrl.Result{}, fmt.Errorf("Fail to add DeviceService to EdgeX: %v", err)
	}
	log.V(4).Info("Successfully add DeviceService to EdgeX",
		"DeviceService", ds.GetName(), "EdgeXId", dsEdgeXId)
	ds.Spec.Id = dsEdgeXId
	ds.Status.AddedToEdgeX = true

	return ctrl.Result{}, r.Update(ctx, &ds)
}
```

#### valuedescriptor_controller

```go
func (r *ValueDescriptorReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := r.Log.WithValues("valuedescriptor", req.NamespacedName)
	var vd devicev1alpha1.ValueDescriptor
    // 先在k8s资源里查找这个valuedescriptor
	if err := r.Get(ctx, req.NamespacedName, &vd); err != nil {
		return ctrl.Result{}, err
	}
	// 1. check if the Edgex code-data has the corresponding ValueDescriptor
	// NOTE this version does not support valuedescriptor update
    // 再去EdgeX中查找这个valuedescriptor
	_, err := r.GetValueDescriptorByName(vd.GetName())
	if err == nil {
        // 如果找到了
		log.Info("ValueDescriptor already exists on EdgeX")
		return ctrl.Result{}, nil
	}
	if !clis.IsNotFoundErr(err) {
		log.Info("Fail to visit the Edgex core-data-service")
		return ctrl.Result{}, nil
	}

	// 2. create one if the ValueDescriptor doesnot exist
    // 如果没有找到，就创建一个后，添加到EdgeX中
	edgexId, err := r.AddValueDescript(toEdgexValue(vd))
	if err != nil {
		return ctrl.Result{}, fmt.Errorf("Fail to add ValueDescriptor to Edgex: %v", err)
	}
	log.V(4).Info("Successfully add ValueDescriptor to Edgex",
		"ValueDescriptor", vd.GetName(), "EdgexId", edgexId)
	vd.Spec.Id = edgexId
	vd.Status.AddedToEdgeX = true

	return ctrl.Result{}, r.Update(ctx, &vd)
}
```

### Syncer

#### device_syncer

```go
func (ds *DeviceSyncer) Run(stop <-chan struct{}) {
	ds.log.Info("starting the DeviceSyncer...")
	go func() {
		for {
			<-time.After(ds.syncPeriod)
            //每次到了同步时间
			// list devices on edgex foundry
            // 调用ListDevices()，其实是DeviceSyncer内的CoreMetaClient的功能来获取edgex上的devices
			eDevs, err := ds.ListDevices()
			if err != nil {
				ds.log.Error(err, "fail to list the devices object on the EdgeX Foundry")
				continue
			}
			// list devices on Kubernetes
            // 来获取kubernetes上的device列表
			var kDevs devv1.DeviceList
			if err := ds.List(context.TODO(), &kDevs); err != nil {
				ds.log.Error(err, "fail to list the devices object on the Kubernetes")
				continue
			}
			// create the devices on Kubernetes but not on EdgeX
            // 找到所有在kubernetes上，但不在EdgeX上的device的device
			newKDevs := findNewDevices(eDevs, kDevs.Items)
			if len(newKDevs) != 0 {
                // 在EdgeX上创建新的device
				if err := createDevices(ds.log, ds.Client, newKDevs); err != nil {
					ds.log.Error(err, "fail to create devices")
					continue
				}
			}
			ds.log.V(5).Info("new devices not found")
		}
	}()

	<-stop
	ds.log.Info("stopping the device syncer")
}
```

#### deviceprofile_syncer

```go
func (ds *DeviceProfileSyncer) Run(stop <-chan struct{}) {
	ds.log.Info("starting the DeviceProfileSyncer...")
	go func() {
		for {
			<-time.After(ds.syncPeriod)
			// list device profiles on edgex foundry
			eDevs, err := ds.ListDeviceProfile()
			if err != nil {
				ds.log.Error(err, "fail to list the deviceprofile object on the EdgeX Foundry")
				continue
			}
			// list device profiles on Kubernetes
			var kDevs devv1.DeviceProfileList
			if err := ds.List(context.TODO(), &kDevs); err != nil {
				ds.log.Error(err, "fail to list the deviceprofile object on the Kubernetes")
				continue
			}
			// create the devices on Kubernetes but not on EdgeX
			newKDevs := findNewDeviceProfile(eDevs, kDevs.Items)
			if len(newKDevs) != 0 {
				if err := createDeviceProfile(ds.log, ds.Client, newKDevs); err != nil {
					ds.log.Error(err, "fail to create devices profile")
					continue
				}
			}
			ds.log.V(5).Info("new deviceprofile not found")
		}
	}()

	<-stop
	ds.log.Info("stopping the deviceprofile syncer")
}
```

#### deviceservice_syncer

```go
func (ds *DeviceServiceSyncer) Run(stop <-chan struct{}) {
	ds.log.Info("starting the DeviceServiceSyncer...")
	go func() {
		for {
			<-time.After(ds.syncPeriod)
			// list deviceservice on edgex foundry
			eDevs, err := ds.ListDeviceServices()
			if err != nil {
				ds.log.Error(err, "fail to list the deviceservice object on the EdgeX Foundry")
				continue
			}
			// list deviceservice on Kubernetes
			var kDevs devv1.DeviceServiceList
			if err := ds.List(context.TODO(), &kDevs); err != nil {
				ds.log.Error(err, "fail to list the deviceservice object on the Kubernetes")
				continue
			}
			// create the deviceservice on Kubernetes but not on EdgeX
			newKDevs := findNewDeviceService(eDevs, kDevs.Items)
			if len(newKDevs) != 0 {
				if err := createDeviceService(ds.log, ds.Client, newKDevs); err != nil {
					ds.log.Error(err, "fail to create deviceservice")
					continue
				}
			}
			ds.log.V(5).Info("new deviceservice not found")
		}
	}()

	<-stop
	ds.log.Info("stopping the device syncer")
}

```

## 2021/9/7

### Crd修改

> DeviceProfileSpec删除EdgeXLabels，添加NodePool、Labels

```go
type DeviceProfileSpec struct {
	// NodePool specifies which nodePool the deviceProfile belongs to
	NodePool    string `json:"nodePool,omitempty"`
	Description string `json:"description,omitempty"`
	// Manufacturer of the device
	Manufacturer string `json:"manufacturer,omitempty"`
	// Model of the device
	Model string `json:"model,omitempty"`
	// Labels used to search for groups of profiles on EdgeX Foundry
	Labels          []string         `json:"labels,omitempty"`
	DeviceResources []DeviceResource `json:"deviceResources,omitempty"`

	// TODO support the following field
	DeviceCommands []ProfileResource `json:"deviceCommands,omitempty"`
	CoreCommands   []Command         `json:"coreCommands,omitempty"`
}
```

> 修改DeviceProfileStatus为EdgeId、Synced两个属性

```go
type DeviceProfileStatus struct {
	EdgeId string `json:"id,omitempty"`
	Synced bool   `json:"Synced,omitempty"`
}
```

### deviceprofile_client

```go
// 这个定义了这个client的地址、端口
type EdgexDeviceProfile struct {
	*resty.Client
	Host string
	Port int
	logr.Logger
}

const (
    // 定义地址和EdgeXObject的名称
	DeviceProfilePath = "/api/v1/deviceprofile"
	EdgeXObjectName   = "device-controller/edgex-object.name"
)

// 新建一个EdgeexDeviceProfile
func NewEdgexDeviceProfile(host string, port int, log logr.Logger) *EdgexDeviceProfile {
	return &EdgexDeviceProfile{
        // Client就是一个http的服务器
		Client: resty.New(),
		Host:   host,
		Port:   port,
		Logger: log,
	}
}

func getListDeviceProfileURL(host string, port int, opts devcli.ListOptions) (string, error) {
	url := fmt.Sprintf("http://%s:%d%s", host, port, DeviceProfilePath)
    // 填充url
	if len(opts.LabelSelector) > 1 {
		return url, fmt.Errorf("Multiple labels: list only support one label")
	}
    // 这里是不是写错了？
	if len(opts.LabelSelector) > 0 && len(opts.LabelSelector) > 0 {
		return url, fmt.Errorf("Multi list options: list action can't use 'label' with 'manufacturer' or 'model'")
	}
	for _, v := range opts.LabelSelector {
		url = fmt.Sprintf("%s/label/%s", url, v)
	}
	// 判断parameters是否满足条件
	listParameters := []string{"manufacturer", "model"}
	for k, v := range opts.FieldSelector {
		if !strutil.IsInStringLst(listParameters, k) {
			return url, fmt.Errorf("Invaild list options: %s", k)
		}
		url = fmt.Sprintf("%s/%s/%s", url, k, v)
	}
	return url, nil
}

// 来获取EdgexDeviceProfile的列表
func (cdc *EdgexDeviceProfile) List(ctx context.Context, opts devcli.ListOptions) ([]v1alpha1.DeviceProfile, error) {
	cdc.V(5).Info("will list DeviceProfiles")
    // 这个是在获取DeviceProfile的整个列表的 访问地址，即，在EdgeX上面获取列表
	lp, err := getListDeviceProfileURL(cdc.Host, cdc.Port, opts)
	if err != nil {
		return nil, err
	}
    // 访问这个ip地址
	resp, err := cdc.R().EnableTrace().Get(lp)
	if err != nil {
		return nil, err
	}
    // 将结果放入dps中
	dps := []models.DeviceProfile{}
	if err := json.Unmarshal(resp.Body(), &dps); err != nil {
		return nil, err
	}
   // 这里是将从EdgeX上获得的内容，转化为Kubernetes上的crd内容，再返回
	deviceProfiles := make([]v1alpha1.DeviceProfile, len(dps))
	for i, dp := range dps {
		deviceProfiles[i] = toKubeDeviceProfile(&dp)
	}
	return deviceProfiles, nil
}

func (cdc *EdgexDeviceProfile) Get(ctx context.Context, name string, opts devcli.GetOptions) (*v1alpha1.DeviceProfile, error) {
	cdc.V(5).Info("will get DeviceProfiles", "DeviceProfile", name)
	var dp models.DeviceProfile
    // 构建获取EdgexDeviceProfile的get请求
	getURL := fmt.Sprintf("http://%s:%d%s/name/%s", cdc.Host, cdc.Port, DeviceProfilePath, name)
	resp, err := cdc.R().Get(getURL)
	if err != nil {
		return nil, err
	}
	if string(resp.Body()) == "Item not found\n" {
		return nil, errors.New("Item not found")
	}
	if err = json.Unmarshal(resp.Body(), &dp); err != nil {
		return nil, err
	}
    // 将这个EdgeX对象转换为kubernetes上定义的deviceprofile对象
	kubedp := toKubeDeviceProfile(&dp)
	return &kubedp, nil
}

func (cdc *EdgexDeviceProfile) Create(ctx context.Context, deviceProfile *v1alpha1.DeviceProfile, opts devcli.CreateOptions) (*v1alpha1.DeviceProfile, error) {
    // 首先将kubernetes上的deviceProfile对象转换为EdgeX上的EdgeXDeviceProfile对象
	edgeDp := ToEdgeXDeviceProfile(deviceProfile)
    // 变成json格式
	dpJson, err := json.Marshal(edgeDp)
	if err != nil {
		return nil, err
	}
	postURL := fmt.Sprintf("http://%s:%d%s", cdc.Host, cdc.Port, DeviceProfilePath)
	// 发起post请求
    resp, err := cdc.R().SetBody(dpJson).Post(postURL)
	if err != nil {
		return nil, err
	}
	if resp.StatusCode() != http.StatusOK {
		return nil, fmt.Errorf("create edgex deviceProfile err: %s", string(resp.Body())) 
        // 假定 resp.Body() 存了 msg 信息
	}
	deviceProfile.Status.EdgeId = string(resp.Body())
	deviceProfile.Status.Synced = true
    // 设置对应的status
	return deviceProfile, err
}

// TODO
func (cdc *EdgexDeviceProfile) Update(ctx context.Context, deviceProfile *v1alpha1.DeviceProfile, opts devcli.UpdateOptions) (*v1alpha1.DeviceProfile, error) {
	return nil, nil
}

func (cdc *EdgexDeviceProfile) Delete(ctx context.Context, name string, opts devcli.DeleteOptions) error {
	cdc.V(5).Info("will delete the DeviceProfile", "DeviceProfile", name)
	delURL := fmt.Sprintf("http://%s:%d%s/name/%s", cdc.Host, cdc.Port, DeviceProfilePath, name)
	resp, err := cdc.R().Delete(delURL)
	if err != nil {
		return err
	}
	if resp.StatusCode() != http.StatusOK {
		return fmt.Errorf("delete edgex deviceProfile err: %s", string(resp.Body())) 
        // 假定 resp.Body() 存了 msg 信息
	}
	return nil
}
```

### deviceprofile_controller

```go
// DeviceProfileReconciler reconciles a DeviceProfile object
type DeviceProfileReconciler struct {
	client.Client
	Log        logr.Logger
	Scheme     *runtime.Scheme
	edgeClient devcli.DeviceProfileInterface
	NodePool   string
}

//+kubebuilder:rbac:groups=device.openyurt.io,resources=deviceprofiles,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=device.openyurt.io,resources=deviceprofiles/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=device.openyurt.io,resources=deviceprofiles/finalizers,verbs=update

// Reconcile make changes to a deviceprofile object in EdgeX based on it in Kubernetes
func (r *DeviceProfileReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := r.Log.WithValues("deviceprofile", req.NamespacedName)
	var curdp devicev1alpha1.DeviceProfile
    // 首先在Kubernetes中获取当前的deviceprofile
	if err := r.Get(ctx, req.NamespacedName, &curdp); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}
    // 如果获取的deviceprofile的NodePool与控制器自己的NodePool不同的话，就直接返回
	if curdp.Spec.NodePool != r.NodePool {
		return ctrl.Result{}, nil
	}

	dpName := util.GetEdgeNameTrimNodePool(curdp.GetName(), r.NodePool)
    // 获取这个deviceprofile的名称
	var prevdp devicev1alpha1.DeviceProfile
	var exist bool
	edps, err := r.edgeClient.List(context.Background(), devcli.ListOptions{})
	if err != nil {
		return ctrl.Result{}, err
	}
    // 到EdgeX找这个名称的deviceprofile
	for _, edp := range edps {
		if strings.ToLower(edp.Name) == dpName {
			prevdp = edp
			exist = true
			break
		}
	}
	
	if !curdp.ObjectMeta.DeletionTimestamp.IsZero() {
        // 如果k8s上的这个deviceprofile到了删除时间
		if exist {
            // 如果在EdgeX上也存在，就把EdgeX上的这个删除
			if err := r.edgeClient.Delete(context.Background(), prevdp.Name, devcli.DeleteOptions{}); err != nil {
				return ctrl.Result{}, fmt.Errorf("Fail to delete DeviceProfile on Edgex: %v", err)
			}
			log.Info("Successfully delete DeviceProfile on EdgeX", "DeviceProfile", prevdp.Name)
		}
        // 删掉Finalizer,使得controller可以把他删掉
		controllerutil.RemoveFinalizer(&curdp, "devicecontroller.openyurt.io")
		err := r.Update(context.TODO(), &curdp)

		return ctrl.Result{}, err
	}
	// 如果deviceprofile没有这个finalizer的话
	if !controllerutil.ContainsFinalizer(&curdp, "devicecontroller.openyurt.io") {
        // 添加这个finalizer
		controllerutil.AddFinalizer(&curdp, "devicecontroller.openyurt.io")
		// 更新这个deviceprofile
        if err = r.Update(context.TODO(), &curdp); err != nil {
			if apierrors.IsConflict(err) {
				return ctrl.Result{Requeue: true}, nil
			}
			return ctrl.Result{}, err
		}
	}
    // 如果在EdgeX上不存在的话
	if !exist {
        // 就创建一个
		curdp, err := r.edgeClient.Create(context.Background(), &curdp, devcli.CreateOptions{})
		if err != nil {
			return ctrl.Result{}, fmt.Errorf("Fail to add DeviceProfile to Edgex: %v", err)
		}
		log.Info("Successfully add DeviceProfile to EdgeX",
			"DeviceProfile", curdp.GetName(), "EdgeId", curdp.Status.EdgeId)
		return ctrl.Result{}, r.Status().Update(ctx, curdp)
	}
	curdp.Spec.NodePool = ""
    // 如果k8s上的deviceprofile与实际EdgeX上的deviceprofile不同的话
	if !reflect.DeepEqual(curdp.Spec, prevdp.Spec) {
		// TODO
		log.Info("controller doesn't support update deviceprofile from Kubernetes to EdgeX")
		return ctrl.Result{}, nil
	}
	return ctrl.Result{}, nil
}
```

### deviceprofile_syncer

```go
func (ds *DeviceProfileSyncer) Run(stop <-chan struct{}) {
	ds.log.Info("starting the DeviceProfileSyncer...")
	go func() {
		for {
			<-time.After(ds.syncPeriod)
            // 同步时间到了
			// list devices on edgex foundry
            // 查询在edgex上的deviceprofile列表
			eDevs, err := ds.edgeClient.List(context.Background(), devcli.ListOptions{})
			if err != nil {
				ds.log.Error(err, "fail to list the deviceprofile object on the EdgeX Foundry")
				continue
			}
            // 为他们添加NodePool属性
			addNodePoolField(eDevs, ds.NodePool)
			// list devices on Kubernetes
            // 查询在kubernetes上的deviceprofile列表
			var kDevs devicev1alpha1.DeviceProfileList
			if err := ds.List(context.TODO(), &kDevs); err != nil {
				ds.log.Error(err, "fail to list the deviceprofile object on the Kubernetes")
				continue
			}
			// create the device profiles on Kubernetes but not on EdgeX
            // 找到在kubernetes上但不在EdgeX上的列表
			newKDevs, updateKDevs := findNewUpdateDeviceProfile(eDevs, kDevs.Items)
			// 添加新的内容
            if len(newKDevs) != 0 {
                // 但是这里是不是有问题啊？为什么用k8s的client进行创建？
				if err := createDeviceProfile(ds.log, ds.Client, newKDevs, ds.NodePool); err != nil {
					ds.log.Error(err, "fail to create device profiles")
					continue
				}
			}
			// update the device profiles according EdgeX
            // 更新的工作
			if len(updateKDevs) != 0 {
				// TODO
			}
			// delete the device profiles on Kubernetes but not on Egdex
            // 删除这些在k8s上，但是不在EdgeX上的device profile
			deleteKDevs := findDeleteDeviceProfile(eDevs, kDevs.Items)
			if len(deleteKDevs) != 0 {
				if err := deleteDeviceProfile(ds.log, ds.Client, deleteKDevs); err != nil {
					ds.log.Error(err, "fail to delete device profiles")
				}
			}
		}
	}()

	<-stop
	ds.log.Info("stopping the deviceprofile syncer")
}

func addNodePoolField(edgeXDevs []devicev1alpha1.DeviceProfile, NodePoolName string) {
	// 给每个EdgeX上的deviceprofile添加NodePool的名称
    for i, _ := range edgeXDevs {
		edgeXDevs[i].Spec.NodePool = NodePoolName
	}
}

// findNewUpdateDeviceProfile finds deviceprofiles that have been created on the EdgeX but not the Kubernetes
func findNewUpdateDeviceProfile(edgeXDevs, kubeDevs []devicev1alpha1.DeviceProfile) ([]devicev1alpha1.DeviceProfile, []devicev1alpha1.DeviceProfile) {
	var addDevs, updateDevs []devicev1alpha1.DeviceProfile
    // 新建一个update切片，将所有EdgeX上的状态与Kubernetes上状态不同的deviceprofile放入这个切片中
	for _, exd := range edgeXDevs {
		var exist bool
		for _, kd := range kubeDevs {
			if strings.ToLower(exd.Name) == util.GetEdgeNameTrimNodePool(kd.Name, kd.Spec.NodePool) {
				exist = true
				if !reflect.DeepEqual(exd.Spec, kd.Spec) {
					kd.Spec = exd.Spec
					updateDevs = append(updateDevs, kd)
				}
				break
			}
		}
        // 对于在EdgeX上有，但是Kubernetes上没有的，放入addDevs中
		if !exist {
			addDevs = append(addDevs, exd)
		}
	}
	// 最后返回两个切片，一个是需要在Kubernetes中添加的切片，一个是需要更新的切片
	return addDevs, updateDevs
}

// findDeleteDeviceProfile finds deviceprofiles that exist on the Kubernetes but not on the EdgeX
func findDeleteDeviceProfile(edgeXDevs, kubeDevs []devicev1alpha1.DeviceProfile) []devicev1alpha1.DeviceProfile {
	var deleteDevs []devicev1alpha1.DeviceProfile
	for _, kd := range kubeDevs {
		var exist bool
		for _, exd := range edgeXDevs {
			if strings.ToLower(exd.Name) == util.GetEdgeNameTrimNodePool(kd.Name, kd.Spec.NodePool) {
				exist = true
				break
			}
		}
        // 如果在Kubernetes上存在，但是在EdgeX上不存在，且k8s上对应的Status.Synced状态是true的话
        // 将他们添加到切片
		if !exist && kd.Status.Synced {
			deleteDevs = append(deleteDevs, kd)
		}
	}
	return deleteDevs
}

func getKubeNameWithPrefix(edgeName, NodePoolName string) string {
	if NodePoolName == "" {
		return edgeName
	}
    // 返回NodePoolName-edgeName这种格式
	return fmt.Sprintf("%s-%s", NodePoolName, edgeName)
}

// createDeviceProfile creates the list of device profiles
func createDeviceProfile(log logr.Logger, cli client.Client, edgeXDevs []devicev1alpha1.DeviceProfile, NodePoolName string) error {
	for _, ed := range edgeXDevs {
        // 设置deviceprofile的名称
		ed.SetName(getKubeNameWithPrefix(ed.GetName(), NodePoolName))
        // 在k8s上添加这个deviceprofile
		if err := cli.Create(context.TODO(), &ed); err != nil {
			if apierrors.IsAlreadyExists(err) {
				log.Info("DeviceProfile already exist on Kubernetes", "deviceprofile", strings.ToLower(ed.Name))
				continue
			}
			return err
		}
		if err := cli.Status().Update(context.TODO(), &ed); err != nil {
			return err
		}
		log.Info("Successfully create DeviceProfile to Kubernetes", "DeviceProfile", ed.GetName())
	}
	return nil
}

func deleteDeviceProfile(log logr.Logger, cli client.Client, kubeDevs []devicev1alpha1.DeviceProfile) error {
    // 在kubernetes上删除这些deviceprofile
	for _, kd := range kubeDevs {
		if err := cli.Delete(context.TODO(), &kd); err != nil {
			if apierrors.IsNotFound(err) {
				log.Info("DeviceProfile doesn't exist on Kubernetes", "deviceprofile", kd.Name)
				continue
			}
			return err
		}
		log.Info("Successfully delete DeviceProfile on Kubernetes", "DeviceProfile", kd.GetName())
	}
	return nil
}

```

## 2021/9/21

### Crd修改

> DeviceServiceSpec删除Id、LastConnected、LastReported属性，添加Managed、NodePool属性

```go
type DeviceServiceSpec struct {
	// Information describing the device
	Description string `json:"description,omitempty"`
	// operational state - either enabled or disabled
	OperatingState OperatingState `json:"operatingState,omitempty"`
	// tags or other labels applied to the device service for search or other
	// identification needs on the EdgeX Foundry
	Labels []string `json:"labels,omitempty"`
	// address (MQTT topic, HTTP address, serial bus, etc.) for reaching
	// the service
	Addressable Addressable `json:"addressable,omitempty"`
	// Device Service Admin State
	AdminState AdminState `json:"adminState,omitempty"`
	// True means deviceService is managed by cloud, cloud can update the related fields
	// False means cloud can't update the fields
	Managed bool `json:"managed,omitempty"`
	// NodePool indicates which nodePool the deviceService comes from
	NodePool string `json:"nodePool,omitempty"`
}
```

> DeviceServiceStatus添加Synced、EdgeId、LastConnected、LastReported、AdminState、Conditions字段，并且添加get和set Conditions方法

```go
type DeviceServiceStatus struct {
	// Synced indicates whether the device already exists on both OpenYurt and edge platform
	Synced bool `json:"synced,omitempty"`
	// the Id assigned by the edge platform
	EdgeId string `json:"edgeId,omitempty"`
	// time in milliseconds that the device last reported data to the core
	LastConnected int64 `json:"lastConnected,omitempty"`
	// time in milliseconds that the device last reported data to the core
	LastReported int64 `json:"lastReported,omitempty"`
	// Device Service Admin State
	AdminState AdminState `json:"adminState,omitempty"`
	// current deviceService state
	// +optional
	Conditions clusterv1.Conditions `json:"conditions,omitempty"`
}
func (ds *DeviceService) SetConditions(conditions clusterv1.Conditions) {
	ds.Status.Conditions = conditions
}

func (ds *DeviceService) GetConditions() clusterv1.Conditions {
	return ds.Status.Conditions
}
```

### deviceservice_client

```go
type EdgexDeviceServiceClient struct {
	*resty.Client
	CoreMetaClient ClientURL
	logr.Logger
}

func NewEdgexDeviceServiceClient(coreMetaClient ClientURL, log logr.Logger) *EdgexDeviceServiceClient {
	return &EdgexDeviceServiceClient{
		Client:         resty.New(),
		CoreMetaClient: coreMetaClient,
		Logger:         log,
	}
}

// Create function sends a POST request to EdgeX to add a new deviceService
func (eds *EdgexDeviceServiceClient) Create(ctx context.Context, deviceservice *v1alpha1.DeviceService, options edgeCli.CreateOptions) (*v1alpha1.DeviceService, error) {
    // 将kubernetes上的deviceservice转化为EdgeX上的DevcieServcie
	ds := toEdgexDeviceService(deviceservice)
	eds.V(5).Info("will add the DeviceServices",
		"DeviceService", ds.Name)
	dpJson, err := json.Marshal(&ds)
	if err != nil {
		return nil, err
	}
    // 变成JSON格式，发起POST请求
	postPath := fmt.Sprintf("http://%s:%d%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, DeviceServicePath)
    // 接收返回信息
	resp, err := eds.R().
		SetBody(dpJson).Post(postPath)
	if err != nil {
		return nil, err
	} else if resp.StatusCode() != http.StatusOK {
        // 如果返回结果不是http.StatusOK
		return nil, fmt.Errorf("create deviceService on edgex foundry failed, the response is : %s", resp.Body())
	}
	createdDs := deviceservice.DeepCopy()
    // 修改Status状态
	createdDs.Status.EdgeId = string(resp.Body())
	return createdDs, err
}

// Delete function sends a request to EdgeX to delete a deviceService
func (eds *EdgexDeviceServiceClient) Delete(ctx context.Context, name string, option edgeCli.DeleteOptions) error {
	eds.V(5).Info("will delete the DeviceService",
		"DeviceService", name)
    // 拼出Delete的URL
	delURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, DeviceServicePath, name)
    // 发起删除的请求
	resp, err := eds.R().Delete(delURL)
	if err != nil {
		return err
	}
	if string(resp.Body()) != "true" {
		return errors.New(string(resp.Body()))
	}
	return nil
}

// Update is used to set the admin or operating state of the deviceService by unique name of the deviceService.
// TODO support to update other fields
func (eds *EdgexDeviceServiceClient) Update(ctx context.Context, ds *v1alpha1.DeviceService, options edgeCli.UpdateOptions) (*v1alpha1.DeviceService, error) {
    // 看起来这里主要修改的就是AdminState和OperatingState两个字段的内容
    // 根据deviceservice找到deviceservice的名称
	actualDSName := getEdgeDeviceServiceName(ds)
    // 生成需要put的url
	putBaseURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, DeviceServicePath, actualDSName)
	if ds == nil {
		return nil, nil
	}
    // 如果需要更新的devicesevice的spec下的adminstate不为空
	if ds.Spec.AdminState != "" {
        // 拼接出请求信息
		amURL := fmt.Sprintf("%s/adminstate/%s", putBaseURL, ds.Spec.AdminState)
        // 发出Put请求
		if rep, err := resty.New().R().SetHeader("Content-Type", "application/json").Put(amURL); err != nil {
			return nil, err
		} else if rep.StatusCode() != http.StatusOK {
			return nil, fmt.Errorf("failed to update deviceService: %s, get response: %s", actualDSName, string(rep.Body()))
		}
	}
    // 如果需要更新的deviceservcie的OperatingState不为空
	if ds.Spec.OperatingState != "" {
        // 拼接出请求url
		opURL := fmt.Sprintf("%s/opstate/%s", putBaseURL, ds.Spec.OperatingState)
        // 发起put请求
		if rep, err := resty.New().R().
			SetHeader("Content-Type", "application/json").Put(opURL); err != nil {
			return nil, err
		} else if rep.StatusCode() != http.StatusOK {
			return nil, fmt.Errorf("failed to update deviceService: %s, get response: %s", actualDSName, string(rep.Body()))
		}
	}

	return ds, nil
}

// Get is used to query the deviceService information corresponding to the deviceService name
func (eds *EdgexDeviceServiceClient) Get(ctx context.Context, name string, options edgeCli.GetOptions) (*v1alpha1.DeviceService, error) {
	eds.V(5).Info("will get DeviceServices",
		"DeviceService", name)
	var ds v1alpha1.DeviceService
    // 拼接出查询DeviceService的URL
	getURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, DeviceServicePath, name)
	resp, err := eds.R().Get(getURL)
	if err != nil {
		return &ds, err
	}
    // 如果没有找到
	if string(resp.Body()) == "Item not found\n" ||
		strings.HasPrefix(string(resp.Body()), "no item found") {
		return &ds, errors.New("Item not found")
	}
    // 找到以后反序列化整体内容
	var dp models.DeviceService
	err = json.Unmarshal(resp.Body(), &dp)
    // 将找到的dp转换为kubernetes上面的deviceservice
	ds = toKubeDeviceService(dp)
	return &ds, err
}

// List is used to get all deviceService objects on edge platform
// The Hanoi version currently supports only a single label and does not support other filters
func (eds *EdgexDeviceServiceClient) List(ctx context.Context, options edgeCli.ListOptions) ([]v1alpha1.DeviceService, error) {
	eds.V(5).Info("will list DeviceServices")
    // 拼接出查询deviceservice列表的url
	lp := fmt.Sprintf("http://%s:%d%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, DeviceServicePath)
	if options.LabelSelector != nil {
        // 如果有LabelSelector，那么在查询url中添加字段
		if _, ok := options.LabelSelector["label"]; ok {
			lp = strings.Join([]string{lp, strings.Join([]string{"label", options.LabelSelector["label"]}, "/")}, "/")
		}
	}
    // 发起get请求，向对应的地址来获取内容
	resp, err := eds.R().
		EnableTrace().
		Get(lp)
	if err != nil {
		return nil, err
	}
	dss := []models.DeviceService{}
    // 将获取到的内容转化为edgex上的deviceservice内容
	if err := json.Unmarshal(resp.Body(), &dss); err != nil {
		return nil, err
	}
	var res []v1alpha1.DeviceService
    // 将这些内容转化为kubernetes上的内容，最终返回
	for _, ds := range dss {
		res = append(res, toKubeDeviceService(ds))
	}
	return res, nil
}

// CreateAddressable function sends a POST request to EdgeX to add a new addressable
func (eds *EdgexDeviceServiceClient) CreateAddressable(ctx context.Context, addressable *v1alpha1.Addressable, options edgeCli.CreateOptions) (*v1alpha1.Addressable, error) {
    // 将传入的addressable转化为EdgeX上的Addressable格式
	as := toEdgeXAddressable(addressable)
	eds.V(5).Info("will add the Addressables",
		"Addressable", as.Name)
    // 将EdgeX上的addressable转为JSON格式
	dpJson, err := json.Marshal(&as)
	if err != nil {
		return nil, err
	}
    // 构建出POST请求的url链接
	postPath := fmt.Sprintf("http://%s:%d%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, AddressablePath)
	// 发起POST请求
    resp, err := eds.R().
		SetBody(dpJson).Post(postPath)
	if err != nil {
		return nil, err
	}
    // 复制addressable，并且将结果写入复制后得到的createdAddr的Id中
	createdAddr := addressable.DeepCopy()
	createdAddr.Id = string(resp.Body())
	return createdAddr, err
}

// DeleteAddressable function sends a request to EdgeX to delete a addressable
func (eds *EdgexDeviceServiceClient) DeleteAddressable(ctx context.Context, name string, options edgeCli.DeleteOptions) error {
	eds.V(5).Info("will delete the Addressable",
		"Addressable", name)
    // 拼接出删除Addressable的url地址
	delURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, AddressablePath, name)
	// 对删除地址发起删除请求
    resp, err := eds.R().Delete(delURL)
	if err != nil {
		return err
	}
	if string(resp.Body()) != "true" {
		return errors.New(string(resp.Body()))
	}
	return nil
}

// UpdateAddressable is used to update the addressable on edgex foundry
func (eds *EdgexDeviceServiceClient) UpdateAddressable(ctx context.Context, device *v1alpha1.Addressable, options edgeCli.UpdateOptions) (*v1alpha1.Addressable, error) {
    // 看来这个版本里还没完成
	return nil, nil
}

// GetAddressable is used to query the addressable information corresponding to the addressable name
func (eds *EdgexDeviceServiceClient) GetAddressable(ctx context.Context, name string, options edgeCli.GetOptions) (*v1alpha1.Addressable, error) {
	eds.V(5).Info("will get Addressables",
		"Addressable", name)
	var addressable v1alpha1.Addressable
    // 拼接出获取这个Addressable的Get请求
	getURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, AddressablePath, name)
	resp, err := eds.R().Get(getURL)
	if err != nil {
		return &addressable, err
	}
    // 看返回结果是否没有找到
	if string(resp.Body()) == "Item not found\n" {
		return &addressable, errors.New("Item not found")
	}
	var maddr models.Addressable
    // 将返回结果转换为EdgeX上的Addressable类型
	err = json.Unmarshal(resp.Body(), &maddr)
    // 再将EdgeX上的Addressable类型转换为Kubernetes上的EdgeX格式内容
	addressable = toKubeAddressable(maddr)
	return &addressable, err
}

// ListAddressables is used to get all addressable objects on edge platform
func (eds *EdgexDeviceServiceClient) ListAddressables(ctx context.Context, options edgeCli.ListOptions) ([]v1alpha1.Addressable, error) {
	eds.V(5).Info("will list Addressables")
    // 拼接获取查询Addressable列表的地址
	lp := fmt.Sprintf("http://%s:%d%s",
		eds.CoreMetaClient.Host, eds.CoreMetaClient.Port, AddressablePath)
    // 对这个地址发起请求
	resp, err := eds.R().
		EnableTrace().
		Get(lp)
	if err != nil {
		return nil, err
	}
    // 将结果写入EdgeX上的Addressable结构体中
	ass := []models.Addressable{}
	if err := json.Unmarshal(resp.Body(), &ass); err != nil {
		return nil, err
	}
    // 将EdgeX上的Addressable转换为Kubernetes上的Addressable
	var res []v1alpha1.Addressable
	for i := range ass {
		res = append(res, toKubeAddressable(ass[i]))
	}
	return res, nil
}
```

### deviceservice_controller

```go
func (r *DeviceServiceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := r.Log.WithValues("deviceService", req.NamespacedName)
	var ds devicev1alpha1.DeviceService
    // 首先，在Kubernetes中找到这个DeviceService
	if err := r.Get(ctx, req.NamespacedName, &ds); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// If objects doesn't belong to the edge platform to which the controller is connected, the controller does not handle events for that object
    // 如果对象不属于控制器所连接的边缘平台，则控制器不处理该对象的事件
	if ds.Spec.NodePool != r.NodePool {
		return ctrl.Result{}, nil
	}
	log.V(4).Info("Reconciling the DeviceService object", "DeviceService", ds.GetName())
	// Update deviceService conditions
    // 更新deviceService的状态
	defer func() {
		conditions.SetSummary(&ds,
			conditions.WithConditions(
				devicev1alpha1.DeviceServiceSyncedCondition, devicev1alpha1.DeviceServiceManagingCondition),
		)
		err := r.Status().Update(ctx, &ds)
		if client.IgnoreNotFound(err) != nil {
			log.Error(err, "update deviceService conditions failed", "deviceService")
		}
	}()

	// 1. Handle the deviceService deletion event
    // 处理deviceService的删除事件
	if err := r.reconcileDeleteDeviceService(ctx, &ds); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	} else if !ds.ObjectMeta.DeletionTimestamp.IsZero() {
		return ctrl.Result{}, nil
	}

	if ds.Status.Synced == false {
        // 如果deviceservice的synced状态是false的话，就要创建这个deviceservice
		// 2. Synchronize OpenYurt deviceService to edge platform
		if err := r.reconcileCreateDeviceService(ctx, &ds, log); err != nil {
			if apierrors.IsConflict(err) {
				return ctrl.Result{Requeue: true}, nil
			} else {
				return ctrl.Result{}, err
			}
		}
	} else if ds.Spec.Managed == true {
		// 3. If the deviceService has been synchronized and is managed by the cloud, reconcile the deviceService fields
        // 如果这个deviceservice已经被同步了，且被云端管理，就要更新这个deviceservice
		if err := r.reconcileUpdateDeviceService(ctx, &ds, log); err != nil {
			if apierrors.IsConflict(err) {
				return ctrl.Result{Requeue: true}, nil
			} else {
				return ctrl.Result{}, err
			}
		}
	}
	return ctrl.Result{}, nil
}
```

```go
func (r *DeviceServiceReconciler) SetupWithManager(mgr ctrl.Manager) error {
	coreMetaCliInfo := edgexCli.ClientURL{Host: "edgex-core-metadata", Port: 48081}
	r.deviceServiceCli = edgexCli.NewEdgexDeviceServiceClient(coreMetaCliInfo, r.Log)

	nodePool, err := util.GetNodePool(mgr.GetConfig())
	if err != nil {
		return err
	}
	r.NodePool = nodePool

	// register the filter field for deviceService
    // 这里的filter field还不懂是什么意思
	if err := mgr.GetFieldIndexer().IndexField(context.TODO(), &devicev1alpha1.DeviceService{}, "spec.nodePool", func(rawObj client.Object) []string {
		deviceService := rawObj.(*devicev1alpha1.DeviceService)
		return []string{deviceService.Spec.NodePool}
	}); err != nil {
		return err
	}
	return ctrl.NewControllerManagedBy(mgr).
		For(&devicev1alpha1.DeviceService{}).
		Complete(r)
}
```

```go
func (r *DeviceServiceReconciler) reconcileDeleteDeviceService(ctx context.Context, ds *devicev1alpha1.DeviceService) error {
	// gets the actual name of deviceService on the edge platform from the Label of the device
    // 从设备的Label中获取边缘平台上deviceService的实际名称
	edgeDeviceServiceName := ds.ObjectMeta.Labels[EdgeXObjectName]
	if ds.ObjectMeta.DeletionTimestamp.IsZero() {
        // 这个应用还没被删除
        // 这里要注意，如果DeletionTimestam不为0的话就是说明正在删除
		if len(ds.GetFinalizers()) == 0 {
			patchString := map[string]interface{}{
				"metadata": map[string]interface{}{
					"finalizers": []string{devicev1alpha1.DeviceServiceFinalizer},
				},
			}
            // 如果没有写好finalizers，那么就新建一个finalizers，用patch的方式发布到k8s上
			if patchData, err := json.Marshal(patchString); err != nil {
				return err
			} else {
				if err = r.Patch(ctx, ds, client.RawPatch(types.MergePatchType, patchData)); err != nil {
					return err
				}
			}
		}
	} else {
        // 如果是!ds.ObjectMeta.DeletionTimestamp.IsZero()的意思就是要删除这个对象
        // 这一步是为了去掉finalizers
		patchString := map[string]interface{}{
			"metadata": map[string]interface{}{
				"finalizers": []string{},
			},
		}
		// delete the deviceService in OpenYurt
        // 先去掉这个deviceservice的finalizers，保证他可以被删除了，就会被系统回收了
		if patchData, err := json.Marshal(patchString); err != nil {
			return err
		} else {
			if err = r.Patch(ctx, ds, client.RawPatch(types.MergePatchType, patchData)); err != nil {
				return err
			}
		}
		
		// delete the deviceService object on edge platform
        // 调用写好的Delete函数，发请求将EdgeX上的这个deviceservice进行删除
		err := r.deviceServiceCli.Delete(nil, edgeDeviceServiceName, edgeInterface.DeleteOptions{})
		if err != nil && !clis.IsNotFoundErr(err) {
			return err
		}
	}
	return nil
}
```

```go
func (r *DeviceServiceReconciler) reconcileCreateDeviceService(ctx context.Context, ds *devicev1alpha1.DeviceService, log logr.Logger) error {
	// get the actual name of deviceService on the Edge platform from the Label of the device
	edgeDeviceName := ds.ObjectMeta.Labels[EdgeXObjectName]
	log.V(4).Info("Checking if deviceService already exist on the edge platform", "deviceService", ds.GetName())
	// Checking if deviceService already exist on the edge platform
    // 判断这个deviceService是不是已经在EdgeX平台上了
	if edgeDs, err := r.deviceServiceCli.Get(nil, edgeDeviceName, edgeInterface.GetOptions{}); err != nil {
		if !clis.IsNotFoundErr(err) {
            // 如果这个不是IsNotFoundErr的话就要立马退出
			log.V(4).Error(err, "fail to visit the edge platform")
			return nil
		}
	} else {
		// a. If object exists, the status of the device on OpenYurt is updated
        // 到这里说明这个对象是存在的，那么我们要对这个对象进行更新
		log.V(4).Info("DeviceService already exists on edge platform")
        // 更新他的同步的状态
		ds.Status.Synced = true
		ds.Status.EdgeId = edgeDs.Status.EdgeId
		return r.Status().Update(ctx, ds)
	}

	// b. If object does not exist, a request is sent to the edge platform to create a new deviceService and related addressable
    // 如果这个对象不存在，那就需要向EdgeX平台发起请求来创建一个deviceService和相关的addressable
	addressable := ds.Spec.Addressable
    // 先看这个addressable是否存在
	as, err := r.deviceServiceCli.GetAddressable(nil, addressable.Name, edgeInterface.GetOptions{})
	if err == nil {
        // 存在了
		log.V(4).Info("Addressable already exists on edge platform")
		ds.Spec.Addressable = *as
	} else if clis.IsNotFoundErr(err) {
        // 出错了，但是是不存在的错误，就要创建一个addressable
        // 在EdgeX上创建一个与addressable相同的Addressable
		createdAddr, err := r.deviceServiceCli.CreateAddressable(nil, &addressable, edgeInterface.CreateOptions{})
		if err != nil {
			conditions.MarkFalse(ds, devicev1alpha1.DeviceServiceSyncedCondition, "failed to add addressable to EdgeX", clusterv1.ConditionSeverityWarning, err.Error())
			return fmt.Errorf("failed to add addressable to edge platform: %v", err)
		}
		log.V(4).Info("Successfully add the Addressable to edge platform",
			"Addressable", addressable.Name, "EdgeId", createdAddr.Id)
        // 返回的这个createdAddr其实就是addressable加了edgexid和更新了同步状态而已
		ds.Spec.Addressable.Id = createdAddr.Id
	} else {
        // 否则就是其他错误了，就要停止了
		log.V(4).Error(err, "fail to visit the edge platform core-metatdata-service")
		conditions.MarkFalse(ds, devicev1alpha1.DeviceServiceSyncedCondition, "failed to visit the EdgeX core-metadata-service", clusterv1.ConditionSeverityWarning, err.Error())
		return err
	}
    // 将创建/更新好的addressable进行更新同步
	if err = r.Update(ctx, ds); err != nil {
		return err
	}
	// 在EdgeX平台创建deviceservice
	createdDs, err := r.deviceServiceCli.Create(nil, ds, edgeInterface.CreateOptions{})
	if err != nil {
		log.V(4).Error(err, "failed to create deviceService on edge platform")
		conditions.MarkFalse(ds, devicev1alpha1.DeviceServiceSyncedCondition, "failed to add DeviceService to EdgeX", clusterv1.ConditionSeverityWarning, err.Error())
		return fmt.Errorf("fail to add DeviceService to edge platform: %v", err)
	}

	log.V(4).Info("Successfully add DeviceService to Edge Platform",
		"DeviceService", ds.GetName(), "EdgeId", createdDs.Status.EdgeId)
	ds.Status.EdgeId = createdDs.Status.EdgeId
	ds.Status.Synced = true
	conditions.MarkTrue(ds, devicev1alpha1.DeviceServiceSyncedCondition)
	return r.Status().Update(ctx, ds)
}
```

```go
func (r *DeviceServiceReconciler) reconcileUpdateDeviceService(ctx context.Context, ds *devicev1alpha1.DeviceService, log logr.Logger) error {
	// 1. reconciling the AdminState field of deviceService
    // 获取deviceservice的状态
	newDeviceServiceStatus := ds.Status.DeepCopy()
    // 获取要更新的deviceservice内容
	updateDeviceService := ds.DeepCopy()
	// do not update deviceService's OperatingState
    // 不要更新deviceservice的OperatingState
	updateDeviceService.Spec.OperatingState = ""
	
    // 如果spec的adminstate不为空，且与status状态不同的话
    // adminstate新的状态要写成旧的预期状态
	if ds.Spec.AdminState != "" && ds.Spec.AdminState != ds.Status.AdminState {
		newDeviceServiceStatus.AdminState = ds.Spec.AdminState
	} else {
		updateDeviceService.Spec.AdminState = ""
	}
	// 更新EdgeX上的adminstate状态
	_, err := r.deviceServiceCli.Update(nil, updateDeviceService, edgeInterface.UpdateOptions{})
	if err != nil {
		conditions.MarkFalse(ds, devicev1alpha1.DeviceServiceManagingCondition, "failed to update AdminState of deviceService on edge platform", clusterv1.ConditionSeverityWarning, err.Error())
		return err
	}

	// 2. update the device status on OpenYurt
    // 更新kubernetes上的device的状态
	ds.Status = *newDeviceServiceStatus
	if err = r.Status().Update(ctx, ds); err != nil {
		conditions.MarkFalse(ds, devicev1alpha1.DeviceServiceManagingCondition, "failed to update status of deviceService on openyurt", clusterv1.ConditionSeverityWarning, err.Error())
		return err
	}
	conditions.MarkTrue(ds, devicev1alpha1.DeviceServiceManagingCondition)
	return nil
}
```

### deviceservice_syncer

```go
func (ds *DeviceServiceSyncer) Run(stop <-chan struct{}) {
	ds.log.V(1).Info("starting the DeviceServiceSyncer...")
	go func() {
		for {
			<-time.After(ds.syncPeriod)
            // 时间到了，需要进行同步
			// 1. get deviceServices on edge platform and OpenYurt
            // 首先获取所有在EdgeX平台上的deviceservice列表和在kubernetes上的deviceservice列表
			edgeDeviceServices, kubeDeviceServices, err := ds.getAllDeviceServices()
			if err != nil {
				ds.log.V(3).Error(err, "fail to list the deviceServices")
				continue
			}

			// 2. find the deviceServices that need to be synchronized
            // 接着获取需要做更改的map，需要做同步的map
			redundantEdgeDeviceServices, redundantKubeDeviceServices, syncedDeviceServices :=
				ds.findDiffDeviceServices(edgeDeviceServices, kubeDeviceServices)
			ds.log.V(1).Info("The number of deviceServices waiting for synchronization",
				"Edge deviceServices should be added to OpenYurt", len(redundantEdgeDeviceServices),
				"OpenYurt deviceServices that should be deleted", len(redundantKubeDeviceServices),
				"DeviceServices that should be synchronized", len(syncedDeviceServices))

			// 3. create deviceServices on OpenYurt which are exists in edge platform but not in OpenYurt
            // 在kubernetes上创建在EdgeX平台上但是不在kubernetes上的deviceservice
			if err := ds.syncEdgeToKube(redundantEdgeDeviceServices); err != nil {
				ds.log.V(3).Error(err, "fail to create deviceServices on OpenYurt")
				continue
			}

			// 4. delete redundant deviceServices on OpenYurt
            // 删除在kubernetes上存在，但是在EdgeX上不存在的deviceservice
			if err := ds.deleteDeviceServices(redundantKubeDeviceServices); err != nil {
				ds.log.V(3).Error(err, "fail to delete redundant deviceServices on OpenYurt")
				continue
			}

			// 5. update deviceService status on OpenYurt
            // 更新在kubernetes上的状态
			if err := ds.updateDeviceServices(syncedDeviceServices); err != nil {
				ds.log.Error(err, "fail to update deviceServices")
				continue
			}

			ds.log.V(1).Info("One round of DeviceService synchronization is complete")
		}
	}()

	<-stop
	ds.log.V(1).Info("stopping the deviceService syncer")
}

// Get the existing DeviceService on the Edge platform, as well as OpenYurt existing DeviceService
// edgeDeviceServices：map[actualName]DeviceService
// kubeDeviceServices：map[actualName]DeviceService
func (ds *DeviceServiceSyncer) getAllDeviceServices() (
	map[string]devicev1alpha1.DeviceService, map[string]devicev1alpha1.DeviceService, error) {

	edgeDeviceServices := map[string]devicev1alpha1.DeviceService{}
	kubeDeviceServices := map[string]devicev1alpha1.DeviceService{}

	// 1. list deviceServices on edge platform
    // 获取所有在EdgeX平台上的deviceServices
	eDevSs, err := ds.deviceServiceCli.List(nil, iotcli.ListOptions{})
	if err != nil {
		ds.log.V(4).Error(err, "fail to list the deviceServices object on the edge platform")
		return edgeDeviceServices, kubeDeviceServices, err
	}
	// 2. list deviceServices on OpenYurt (filter objects belonging to edgeServer)
    // 获取所有在kubernetes平台上的deviceServices
	var kDevSs devicev1alpha1.DeviceServiceList
	listOptions := client.MatchingFields{"spec.nodePool": ds.NodePool}
    // 找到所有nodePool名称是这个的内容
	if err = ds.List(context.TODO(), &kDevSs, listOptions); err != nil {
		ds.log.V(4).Error(err, "fail to list the deviceServices object on the Kubernetes")
		return edgeDeviceServices, kubeDeviceServices, err
	}
    // 下面两个就是将对应的值放入map对应名称里
	for i := range eDevSs {
		deviceServicesName := eDevSs[i].Labels[EdgeXObjectName]
		edgeDeviceServices[deviceServicesName] = eDevSs[i]
	}

	for i := range kDevSs.Items {
		deviceServicesName := kDevSs.Items[i].Labels[EdgeXObjectName]
		kubeDeviceServices[deviceServicesName] = kDevSs.Items[i]
	}
	return edgeDeviceServices, kubeDeviceServices, nil
}

// Get the list of deviceServices that need to be added, deleted and updated
func (ds *DeviceServiceSyncer) findDiffDeviceServices(
	edgeDeviceService map[string]devicev1alpha1.DeviceService, kubeDeviceService map[string]devicev1alpha1.DeviceService) (
	redundantEdgeDeviceServices map[string]*devicev1alpha1.DeviceService, redundantKubeDeviceServices map[string]*devicev1alpha1.DeviceService, syncedDeviceServices map[string]*devicev1alpha1.DeviceService) {
	// 用来记录所有的增加、删除、更新
	redundantEdgeDeviceServices = map[string]*devicev1alpha1.DeviceService{}
	redundantKubeDeviceServices = map[string]*devicev1alpha1.DeviceService{}
	syncedDeviceServices = map[string]*devicev1alpha1.DeviceService{}
	
    // 先遍历EdgeX上的deviceservice
	for n, v := range edgeDeviceService {
        // 获取名字
		edName := v.Labels[EdgeXObjectName]
		if _, exists := kubeDeviceService[edName]; !exists {
            // 如果这个内容在kubernetes上不存在的话 （新增）
			ed := edgeDeviceService[n]
            // 加入到map中
			redundantEdgeDeviceServices[edName] = ds.completeCreateContent(&ed)
		} else {
            // 如果在kubernetes中存在，那么加入synced的map中（更新）
			kd := kubeDeviceService[edName]
			ed := edgeDeviceService[n]
			syncedDeviceServices[edName] = ds.completeUpdateContent(&kd, &ed)
		}
	}

	for k, v := range kubeDeviceService {
        // 遍历kubernetes中的deviceservice
		if !v.Status.Synced {
			continue
		}
        // 找到Synced状态是true的，然后根据名字，将在k8s但是不在EdgeX上的放入map中
		kdName := v.Labels[EdgeXObjectName]
		if _, exists := edgeDeviceService[kdName]; !exists {
			kd := kubeDeviceService[k]
			redundantKubeDeviceServices[kdName] = &kd
		}
	}
	return
}

// syncEdgeToKube creates deviceServices on OpenYurt which are exists in edge platform but not in OpenYurt
func (ds *DeviceServiceSyncer) syncEdgeToKube(edgeDevs map[string]*devicev1alpha1.DeviceService) error {
	for _, ed := range edgeDevs {
        // 在kubernetes依次创建这些EdgeX列表
		if err := ds.Client.Create(context.TODO(), ed); err != nil {
			if apierrors.IsAlreadyExists(err) {
				ds.log.V(5).Info("DeviceService already exist on Kubernetes",
					"DeviceService", strings.ToLower(ed.Name))
				continue
			}
			ds.log.Info("created deviceService failed:",
				"DeviceService", strings.ToLower(ed.Name))
			return err
		}
	}
	return nil
}

// deleteDeviceServices deletes redundant deviceServices on OpenYurt
func (ds *DeviceServiceSyncer) deleteDeviceServices(redundantKubeDeviceServices map[string]*devicev1alpha1.DeviceService) error {
    // 在kubernetes上删除这些deviceservice列表
	for i := range redundantKubeDeviceServices {
		if err := ds.Client.Delete(context.TODO(), redundantKubeDeviceServices[i]); err != nil {
			ds.log.V(5).Error(err, "fail to delete the DeviceService on Kubernetes",
				"DeviceService", redundantKubeDeviceServices[i].Name)
			return err
		}
	}
	return nil
}

// updateDeviceServicesStatus updates deviceServices status on OpenYurt
func (ds *DeviceServiceSyncer) updateDeviceServices(syncedDeviceServices map[string]*devicev1alpha1.DeviceService) error {
	for _, sd := range syncedDeviceServices {
		if sd.ObjectMeta.ResourceVersion == "" {
			continue
		}
		if err := ds.Client.Status().Update(context.TODO(), sd); err != nil {
			if apierrors.IsConflict(err) {
				ds.log.V(5).Info("update Conflicts",
					"DeviceService", sd.Name)
				continue
			}
			ds.log.V(5).Error(err, "fail to update the DeviceService on Kubernetes",
				"DeviceService", sd.Name)
			return err
		}
	}
	return nil
}
// 下面两个函数的作用是，完善kubernetes与EdgeX平台上deviceservice的转换

// completeCreateContent completes the content of the deviceService which will be created on OpenYurt
func (ds *DeviceServiceSyncer) completeCreateContent(edgeDS *devicev1alpha1.DeviceService) *devicev1alpha1.DeviceService {
	createDevice := edgeDS.DeepCopy()
	createDevice.Spec.NodePool = ds.NodePool
	createDevice.Name = strings.Join([]string{ds.NodePool, createDevice.Name}, "-")
	createDevice.Spec.Managed = false
	return createDevice
}

// completeUpdateContent completes the content of the deviceService which will be updated on OpenYurt
func (ds *DeviceServiceSyncer) completeUpdateContent(kubeDS *devicev1alpha1.DeviceService, edgeDS *devicev1alpha1.DeviceService) *devicev1alpha1.DeviceService {
	updatedDS := kubeDS.DeepCopy()
	// update device status
	updatedDS.Status.LastConnected = edgeDS.Status.LastConnected
	updatedDS.Status.LastReported = edgeDS.Status.LastReported
	updatedDS.Status.AdminState = edgeDS.Status.AdminState
	return updatedDS
}

```

## 2021/9/22

### Crd修改

> DeviceSpec添加了Managed和NodePool两个字段

```go
// DeviceSpec defines the desired state of Device
type DeviceSpec struct {
	// Information describing the device
	Description string `json:"description,omitempty"`
	// Admin state (locked/unlocked)
	AdminState AdminState `json:"adminState,omitempty"`
	// Operating state (enabled/disabled)
	OperatingState OperatingState `json:"operatingState,omitempty"`
	// A map of supported protocols for the given device
	Protocols map[string]ProtocolProperties `json:"protocols,omitempty"`
	// Other labels applied to the device to help with searching
	Labels []string `json:"labels,omitempty"`
	// Device service specific location (interface{} is an empty interface so
	// it can be anything)
	Location string `json:"location,omitempty"`
	// Associated Device Service - One per device
	Service string `json:"service"`
	// Associated Device Profile - Describes the device
	Profile string `json:"profile"`
	// True means device is managed by cloud, cloud can update the related fields
	// False means cloud can't update the fields
	Managed bool `json:"managed,omitempty"`
	// NodePool indicates which nodePool the device comes from
	NodePool string `json:"nodePool,omitempty"`
	// TODO support the following field
	// A list of auto-generated events coming from the device
	// AutoEvents     []AutoEvent                   `json:"autoEvents"`
	// DeviceProperties represents the expected state of the device's properties
	DeviceProperties map[string]DesiredPropertyState `json:"deviceProperties,omitempty"`
}
```

> DeviceStatus中添加Synced、EdgeId、AdminState、OperatingState、Conditions字段

```go
// DeviceStatus defines the observed state of Device
type DeviceStatus struct {
	// Time (milliseconds) that the device last provided any feedback or
	// responded to any request
	LastConnected int64 `json:"lastConnected,omitempty"`
	// Time (milliseconds) that the device reported data to the core
	// microservice
	LastReported int64 `json:"lastReported,omitempty"`
	// Synced indicates whether the device already exists on both OpenYurt and edge platform
	Synced bool `json:"synced,omitempty"`
	// it represents the actual state of the device's properties
	DeviceProperties map[string]ActualPropertyState `json:"deviceProperties,omitempty"`
	EdgeId           string                         `json:"edgeId,omitempty"`
	// Admin state (locked/unlocked)
	AdminState AdminState `json:"adminState,omitempty"`
	// Operating state (enabled/disabled)
	OperatingState OperatingState `json:"operatingState,omitempty"`
	// current device state
	// +optional
	Conditions clusterv1.Conditions `json:"conditions,omitempty"`
}
```

### device_client

```go
// Create function sends a POST request to EdgeX to add a new device
func (efc *EdgexDeviceClient) Create(ctx context.Context, device *devicev1alpha1.Device, options edgeCli.CreateOptions) (*devicev1alpha1.Device, error) {
	dp := toEdgeXDevice(device)
    // 将kubernetes上的device转换为EdgeX上的device
	efc.V(5).Info("will add the Devices",
		"Device", dp.Name)
	dpJson, err := json.Marshal(&dp)
	if err != nil {
		return nil, err
	}
    // 构造post请求
	postPath := fmt.Sprintf("http://%s:%d%s",
		efc.CoreMetaClient.Host, efc.CoreMetaClient.Port, DevicePath)
	resp, err := efc.R().
		SetBody(dpJson).Post(postPath)
	if err != nil {
		return nil, err
	} else if resp.StatusCode() != http.StatusOK {
		return nil, fmt.Errorf("create device on edgex foundry failed, the response is : %s", resp.Body())
	}
	// 返回成功的时候，复制一份k8s上的device
	createdDevice := device.DeepCopy()
    // 为他添加EdgeId字段
	createdDevice.Status.EdgeId = string(resp.Body())
	return createdDevice, err
}

// Delete function sends a request to EdgeX to delete a device
func (efc *EdgexDeviceClient) Delete(ctx context.Context, name string, options edgeCli.DeleteOptions) error {
	efc.V(5).Info("will delete the Device",
		"Device", name)
    // 构造请求的url
	delURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		efc.CoreMetaClient.Host, efc.CoreMetaClient.Port, DevicePath, name)
    // 发起请求
	resp, err := efc.R().Delete(delURL)
	if err != nil {
		return err
	}
	if resp.StatusCode() != http.StatusOK {
		return errors.New(string(resp.Body()))
	}
	return nil
}

// Update is used to set the admin or operating state of the device by unique name of the device.
// TODO support to update other fields
func (efc *EdgexDeviceClient) Update(ctx context.Context, device *devicev1alpha1.Device, options edgeCli.UpdateOptions) (*devicev1alpha1.Device, error) {
	actualDeviceName := getEdgeDeviceName(device)
    // 构造put的url
	putURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		efc.CoreMetaClient.Host, efc.CoreMetaClient.Port, DevicePath, actualDeviceName)
	if device == nil {
		return nil, nil
	}
    // 构造需要修改内容
	updateData := map[string]string{}
	if device.Spec.AdminState != "" {
		updateData["adminState"] = string(device.Spec.AdminState)
	}
	if device.Spec.OperatingState != "" {
		updateData["operatingState"] = string(device.Spec.OperatingState)
	}
	if len(updateData) == 0 {
		return nil, nil
	}
	// 变成json格式
	data, _ := json.Marshal(updateData)
    // 发起更新请求
	rep, err := resty.New().R().
		SetHeader("Content-Type", "application/json").
		SetBody(data).
		Put(putURL)
	if err != nil {
		return nil, err
	} else if rep.StatusCode() != http.StatusOK {
		return nil, fmt.Errorf("failed to update device: %s, get response: %s", actualDeviceName, string(rep.Body()))
	}
	return device, nil
}

// Get is used to query the device information corresponding to the device name
func (efc *EdgexDeviceClient) Get(ctx context.Context, deviceName string, options edgeCli.GetOptions) (*devicev1alpha1.Device, error) {
	efc.V(5).Info("will get Devices",
		"Device", deviceName)
	var device devicev1alpha1.Device
    // 构造get请求地址
	getURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		efc.CoreMetaClient.Host, efc.CoreMetaClient.Port, DevicePath, deviceName)
	resp, err := efc.R().Get(getURL)
	if err != nil {
		return &device, err
	}
    // 如果返回结果是没有找到
	if string(resp.Body()) == "Item not found\n" {
		return &device, errors.New("Item not found")
	}
	var dp models.Device
    // 将结果放入device中
	err = json.Unmarshal(resp.Body(), &dp)
    // 转换为kubernetes上的device
	device = toKubeDevice(dp)
	return &device, err
}

// List is used to get all device objects on edge platform
// The Hanoi version currently supports only a single label and does not support other filters
func (efc *EdgexDeviceClient) List(ctx context.Context, options edgeCli.ListOptions) ([]devicev1alpha1.Device, error) {
    // 拼接获取list列表的url
	lp := fmt.Sprintf("http://%s:%d%s",
		efc.CoreMetaClient.Host, efc.CoreMetaClient.Port, DevicePath)
	// 如果有labelselector的话，就要拼接到url中
    if options.LabelSelector != nil {
		if _, ok := options.LabelSelector["label"]; ok {
			lp = strings.Join([]string{lp, strings.Join([]string{"label", options.LabelSelector["label"]}, "/")}, "/")
		}
	}
    // 发起请求
	resp, err := efc.R().EnableTrace().Get(lp)
	if err != nil {
		return nil, err
	}
    // 获得在EdgeX上的device列表
	dps := []models.Device{}
	if err := json.Unmarshal(resp.Body(), &dps); err != nil {
		return nil, err
	}
    // 将在EdgeX上的device转换为在kubernetes上的device
	var res []devicev1alpha1.Device
	for _, dp := range dps {
		res = append(res, toKubeDevice(dp))
	}
	return res, nil
}

func (efc *EdgexDeviceClient) GetPropertyState(ctx context.Context, propertyName string, d *devicev1alpha1.Device, options edgeCli.GetOptions) (*devicev1alpha1.ActualPropertyState, error) {
    // 获取device的实际名称
	actualDeviceName := getEdgeDeviceName(d)
	// get the old property from status
    // 从状态中获取旧属性（属性名称给了）
	oldAps, exist := d.Status.DeviceProperties[propertyName]
	propertyGetURL := ""
	// 1. query the Get URL of an property
    // 获取查询的get url
	if !exist || (exist && oldAps.GetURL == "") {
        // 如果不存在或者这个属性的geturl为空
        // 根据实际设备名称获取commandRep
		commandRep, err := efc.GetCommandResponseByName(actualDeviceName)
		if err != nil {
			return &devicev1alpha1.ActualPropertyState{}, err
		}
        // 找到属性名称，获取url
		for _, c := range commandRep.Commands {
			if c.Name == propertyName {
				propertyGetURL = c.Get.URL
				break
			}
		}
		if propertyGetURL == "" {
			return nil, fmt.Errorf("this property %s is not exist", propertyName)
		}
	} else {
		propertyGetURL = oldAps.GetURL
	}
	// 2. get the actual property value by the getURL
    // 通过get url来获取实际的属性
	actualPropertyState := devicev1alpha1.ActualPropertyState{
		Name:   propertyName,
		GetURL: propertyGetURL,
	}
    // 这其实就是get请求，加上了一些错误判断
	if resp, err := getPropertyState(propertyGetURL); err != nil {
		return nil, err
	} else {
		var event models.Event
        // 请求结果放入event
		if err := json.Unmarshal(resp.Body(), &event); err != nil {
			return &devicev1alpha1.ActualPropertyState{}, err
		}
        // 从event中取出实际值
		actualPropertyState.ActualValue = getPropertyValueFromEvent(propertyName, event)
	}
	return &actualPropertyState, nil
}

// getPropertyState returns different error messages according to the status code
func getPropertyState(getURL string) (*resty.Response, error) {
    // 这里其实就是对错误进行了一些处理
	resp, err := resty.New().R().Get(getURL)
	if err != nil {
		return resp, err
	}
	if resp.StatusCode() == 400 {
		err = errors.New("request is in an invalid state")
	} else if resp.StatusCode() == 404 {
		err = errors.New("the requested resource does not exist")
	} else if resp.StatusCode() == 423 {
		err = errors.New("the device is locked (AdminState) or down (OperatingState)")
	} else if resp.StatusCode() == 500 {
		err = errors.New("an unexpected error occurred on the server")
	}
	return resp, err
}

func (efc *EdgexDeviceClient) UpdatePropertyState(ctx context.Context, propertyName string, d *devicev1alpha1.Device, options edgeCli.UpdateOptions) error {
	// Get the actual device name
    // 获取设备的实际名称
	acturalDeviceName := getEdgeDeviceName(d)
	
    // 来获取设备的对应属性
	dps := d.Spec.DeviceProperties[propertyName]
	if dps.PutURL == "" {
        // 获取他对应的url
		putURL, err := efc.getPropertyPutURL(acturalDeviceName, dps.Name)
		if err != nil {
			return err
		}
		dps.PutURL = putURL
	}
	// set the device property to desired state
	efc.V(5).Info("setting the property to desired value", "property", dps.Name)
    // 将内容写入后，发起put请求
	rep, err := resty.New().R().
		SetHeader("Content-Type", "application/json").
		SetBody([]byte(fmt.Sprintf(`{"%s": "%s"}`, dps.Name, dps.DesiredValue))).
		Put(dps.PutURL)
	if err != nil {
		return err
	} else if rep.StatusCode() != http.StatusOK {
		return fmt.Errorf("failed to set property: %s, get response: %s", dps.Name, string(rep.Body()))
	} else if rep.Body() != nil {
		// If the parameters are illegal, such as out of range, the 200 status code is also returned, but the description appears in the body
		a := string(rep.Body())
		if strings.Contains(a, "execWriteCmd") {
			return fmt.Errorf("failed to set property: %s, get response: %s", dps.Name, string(rep.Body()))
		}
	}
	return nil
}

// Gets the putURL from edgex foundry which is used to set the device property's value
func (efc *EdgexDeviceClient) getPropertyPutURL(deviceName, cmdName string) (string, error) {
    // 获取所有设备支持的命令
    cr, err := efc.GetCommandResponseByName(deviceName)
	if err != nil {
		return "", err
	}
    // 找到对应的命令就返回这个命令的put url
	for _, c := range cr.Commands {
		if cmdName == c.Name {
			return c.Put.URL, nil
		}
	}
	return "", errors.New("corresponding command is not found")
}

// ListPropertiesState gets all the actual property information about a device
func (efc *EdgexDeviceClient) ListPropertiesState(ctx context.Context, device *devicev1alpha1.Device, options edgeCli.ListOptions) (map[string]devicev1alpha1.DesiredPropertyState, map[string]devicev1alpha1.ActualPropertyState, error) {
    // 获取设备名称
	actualDeviceName := getEdgeDeviceName(device)
	
	dps := map[string]devicev1alpha1.DesiredPropertyState{}
	aps := map[string]devicev1alpha1.ActualPropertyState{}
	cr, err := efc.GetCommandResponseByName(actualDeviceName)
	if err != nil {
		return dps, aps, err
	}

	for _, c := range cr.Commands {
		// DesiredPropertyState only store the basic information and does not set DesiredValue
		resp, err := getPropertyState(c.Get.URL)
		dps[c.Name] = devicev1alpha1.DesiredPropertyState{Name: c.Name, PutURL: c.Put.URL}
		if err != nil {
            // 请求成功
			aps[c.Name] = devicev1alpha1.ActualPropertyState{Name: c.Name, GetURL: c.Get.URL}
		} else {
            // 如果请求失败
			var event models.Event
			if err := json.Unmarshal(resp.Body(), &event); err != nil {
				return dps, aps, err
			}
            // 从返回的event中获取属性实际值
			actualValue := getPropertyValueFromEvent(c.Name, event)
			aps[c.Name] = devicev1alpha1.ActualPropertyState{Name: c.Name, GetURL: c.Get.URL, ActualValue: actualValue}
		}
	}
	return dps, aps, nil
}

// The actual property value is resolved from the returned event
// 实际的属性值，在返回的event中
func getPropertyValueFromEvent(propertyName string, modelEvent models.Event) string {
	actualValue := ""
    // 从event的readings中读取值
	for _, k := range modelEvent.Readings {
		if propertyName == k.Name {
			actualValue = k.Value
			break
		}
	}
	return actualValue
}

// GetCommandResponseByName gets all commands supported by the device
// 获取所有设备支持的命令
func (efc *EdgexDeviceClient) GetCommandResponseByName(deviceName string) (
	models.CommandResponse, error) {
	efc.V(5).Info("will get CommandResponses",
		"CommandResponse", deviceName)

	var vd models.CommandResponse
	getURL := fmt.Sprintf("http://%s:%d%s/name/%s",
		efc.CoreCommandClient.Host, efc.CoreCommandClient.Port, CommandResponsePath, deviceName)

	resp, err := efc.R().Get(getURL)
	if err != nil {
		return vd, err
	}
	if strings.Contains(string(resp.Body()), "Item not found") {
		return vd, errors.New("Item not found")
	}
	err = json.Unmarshal(resp.Body(), &vd)
	return vd, err
}
```

### device_controller

```go
//+kubebuilder:rbac:groups=device.openyurt.io,resources=devices,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=device.openyurt.io,resources=devices/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=device.openyurt.io,resources=devices/finalizers,verbs=update

func (r *DeviceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 根据名称获取这个值（应该是kubernetes上的吧？）
	log := r.Log.WithValues("device", req.NamespacedName)
	var d devicev1alpha1.Device
	if err := r.Get(ctx, req.NamespacedName, &d); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// If objects doesn't belong to the Edge platform to which the controller is connected, the controller does not handle events for that object
    // 如果对象不属于控制器所连接的Edge平台，则控制器不处理该对象的事件
	if d.Spec.NodePool != r.NodePool {
		return ctrl.Result{}, nil
	}

	log.V(4).Info("Reconciling the Device object", "Device", d.GetName())
	// Update the conditions for device
	defer func() {
        // 更新device的conditions
		conditions.SetSummary(&d,
			conditions.WithConditions(devicev1alpha1.DeviceSyncedCondition, devicev1alpha1.DeviceManagingCondition),
		)
		err := r.Status().Update(ctx, &d)
		if client.IgnoreNotFound(err) != nil {
			if !apierrors.IsConflict(err) {
				log.Info("err", "Conditions", d.Status.Conditions)
				log.Error(err, "update device conditions failed")
			}
		}
	}()

	// 1. Handle the device deletion event
    // 处理设备的删除事件
    // 要删除的就删除，不要删除的就增加finalizer
	if err := r.reconcileDeleteDevice(ctx, &d, log); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	} else if !d.ObjectMeta.DeletionTimestamp.IsZero() {
        // 删除了，但是删除时间不为空（正在删除中？）就直接返回
		return ctrl.Result{}, nil
	}

	if d.Status.Synced == false {
        // 如果没有同步
		// 2. Synchronize OpenYurt device objects to edge platform
        // 就要在EdgeX平台创建这个设备对象
		if err := r.reconcileCreateDevice(ctx, &d, log); err != nil {
			if apierrors.IsConflict(err) {
				return ctrl.Result{Requeue: true}, nil
			} else {
				return ctrl.Result{}, err
			}
		}
		return ctrl.Result{}, nil
	} else if d.Spec.Managed == true {
        // 同步了，且被云端管理，就要对这个设备进行更新
		// 3. If the device has been synchronized and is managed by the cloud, reconcile the device properties
		if err := r.reconcileUpdateDevice(ctx, &d, log); err != nil {
			if apierrors.IsConflict(err) {
				return ctrl.Result{RequeueAfter: time.Second * 2}, nil
			}
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *DeviceReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // 这里其实要访问两个微服务，都包装到了deviceCli中了
	coreMetaCliInfo := edgexCli.ClientURL{Host: "edgex-core-metadata", Port: 48081}
	coreCmdCliInfo := edgexCli.ClientURL{Host: "edgex-core-command", Port: 48082}
	r.deviceCli = edgexCli.NewEdgexDeviceClient(coreMetaCliInfo, coreCmdCliInfo, r.Log)

	// Gets the nodePool to which deviceController is deployed
    // 获取deviceController部署到的nodePool
	nodePool, err := util.GetNodePool(mgr.GetConfig())
	if err != nil {
		return err
	}
	r.NodePool = nodePool

	// register the filter field for device
    // 这里的作用一直没有搞懂,filter field到底是干嘛用的？
	if err := mgr.GetFieldIndexer().IndexField(context.TODO(), &devicev1alpha1.Device{}, "spec.nodePool", func(rawObj client.Object) []string {
		device := rawObj.(*devicev1alpha1.Device)
		return []string{device.Spec.NodePool}
	}); err != nil {
		return err
	}
	return ctrl.NewControllerManagedBy(mgr).
		For(&devicev1alpha1.Device{}).
		WithEventFilter(genFirstUpdateFilter("device", r.Log)).
		Complete(r)
}

func (r *DeviceReconciler) reconcileDeleteDevice(ctx context.Context, d *devicev1alpha1.Device, log logr.Logger) error {
	// gets the actual name of the device on the Edge platform from the Label of the device
    // 从设备的label中来获取这个设备在edgex平台上的真实名称
	edgeDeviceName := d.ObjectMeta.Labels[EdgeXObjectName]
	if d.ObjectMeta.DeletionTimestamp.IsZero() {
        // 如果不用删除
		if len(d.GetFinalizers()) == 0 {
            // 如果他的finalizers为空
			patchData, _ := json.Marshal(map[string]interface{}{
				"metadata": map[string]interface{}{
					"finalizers": []string{devicev1alpha1.DeviceFinalizer},
				},
			})
            // 我们就要增加他的finalizer
            // 以patch的方式增加
			if err := r.Patch(ctx, d, client.RawPatch(types.MergePatchType, patchData)); err != nil {
				return err
			}
		}
	} else {
		// delete the device object on the edge platform
        // 如果要删除，就发起删除请求，将内容从EdgeX平台上删除
		err := r.deviceCli.Delete(nil, edgeDeviceName, iotcli.DeleteOptions{})
		if err != nil && !clis.IsNotFoundErr(err) {
			return err
		}

		// delete the device in OpenYurt
        // 然后去掉finalizer，这样以后就可以将kubernetes上的这个内容删除了
		patchData, _ := json.Marshal(map[string]interface{}{
			"metadata": map[string]interface{}{
				"finalizers": []string{},
			},
		})
        // 发请求，做patch修改，其实就是删掉了finalizers
		if err = r.Patch(ctx, d, client.RawPatch(types.MergePatchType, patchData)); err != nil {
			return err
		}
	}
	return nil
}

func (r *DeviceReconciler) reconcileCreateDevice(ctx context.Context, d *devicev1alpha1.Device, log logr.Logger) error {
	// get the actual name of the device on the Edge platform from the Label of the device
    // 从设备的label中获取部署在EdgeX上的设备的真实名称
	edgeDeviceName := d.ObjectMeta.Labels[EdgeXObjectName]
    // 拷贝一份设备的状态
	newDeviceStatus := d.Status.DeepCopy()
	log.V(4).Info("Checking if device already exist on the edge platform", "device", d.GetName())
	// Checking if device already exist on the edge platform
    // 看是否这个设备已经存在在EdgeX平台上了
	edgeDevice, err := r.deviceCli.Get(nil, edgeDeviceName, iotcli.GetOptions{})
	if err == nil {
        // 如果这个对象已经存在，那么就更新这个设备在kubernetes上的状态
		// a. If object exists, the status of the device on OpenYurt is updated
		log.V(4).Info("Device already exists on edge platform")
		newDeviceStatus.EdgeId = edgeDevice.Status.EdgeId
		newDeviceStatus.Synced = true
	} else if clis.IsNotFoundErr(err) {
		// b. If the object does not exist, a request is sent to the edge platform to create a new device
        // 如果这个对象不存在，依靠IsNotFoundErr找到
		log.V(4).Info("Adding device to the edge platform", "device", d.GetName())
        // 就将这个对象创建
		createdEdgeObj, err := r.deviceCli.Create(nil, d, iotcli.CreateOptions{})
		if err != nil {
			conditions.MarkFalse(d, devicev1alpha1.DeviceSyncedCondition, "failed to create device on edge platform", clusterv1.ConditionSeverityWarning, err.Error())
			return fmt.Errorf("fail to add Device to edge platform: %v", err)
		} else {
            // 更新最新的status
			log.V(4).Info("Successfully add Device to edge platform",
				"EdgeDeviceName", edgeDeviceName, "EdgeId", createdEdgeObj.Status.EdgeId)
			newDeviceStatus.EdgeId = createdEdgeObj.Status.EdgeId
			newDeviceStatus.Synced = true
		}
	} else {
        // 如果是其他错误，就直接报错了
		log.V(4).Info("failed to visit the edge platform")
		conditions.MarkFalse(d, devicev1alpha1.DeviceSyncedCondition, "failed to visit the EdgeX core-metadata-service", clusterv1.ConditionSeverityWarning, "")
		return nil
	}
	d.Status = *newDeviceStatus
	conditions.MarkTrue(d, devicev1alpha1.DeviceSyncedCondition)
    // 最后更新设备的状态
	return r.Status().Update(ctx, d)
}

func (r *DeviceReconciler) reconcileUpdateDevice(ctx context.Context, d *devicev1alpha1.Device, log logr.Logger) error {
	// the device has been added to the edge platform, check if each device property are in the desired state
    // 看是否设备的每个状态都在想要的值
	newDeviceStatus := d.Status.DeepCopy()
	// This list is used to hold the names of properties that failed to reconcile
    // 这个列表就是存所有没有调整好的属性的名称
	var failedPropertyNames []string

	// 1. reconciling the AdminState and OperatingState field of device
	log.V(3).Info("reconciling the AdminState and OperatingState field of device")
    // 首先调整adminstate和operatingstate状态
	updateDevice := d.DeepCopy()
	if d.Spec.AdminState != "" && d.Spec.AdminState != d.Status.AdminState {
		newDeviceStatus.AdminState = d.Spec.AdminState
	} else {
		updateDevice.Spec.AdminState = ""
	}

	if d.Spec.OperatingState != "" && d.Spec.OperatingState != d.Status.OperatingState {
		newDeviceStatus.OperatingState = d.Spec.OperatingState
	} else {
		updateDevice.Spec.OperatingState = ""
	}
	_, err := r.deviceCli.Update(nil, updateDevice, iotcli.UpdateOptions{})
	if err != nil {
		conditions.MarkFalse(d, devicev1alpha1.DeviceManagingCondition, "failed to update AdminState or OperatingState of device on edge platform", clusterv1.ConditionSeverityWarning, err.Error())
		return err
	}

	// 2. reconciling the device properties' value
    // 调整设备属性的值
	log.V(3).Info("reconciling the device properties")
	// property updates are made only when the device is operational and unlocked
	if newDeviceStatus.OperatingState == devicev1alpha1.Enabled && newDeviceStatus.AdminState == devicev1alpha1.UnLocked {
        // operatingstate为enable且adminstate为unlock的时候
		newDeviceStatus, failedPropertyNames = r.reconcileDeviceProperties(d, newDeviceStatus, log)
	}
	// 更新状态
	d.Status = *newDeviceStatus

	// 3. update the device status on OpenYurt
    // 更新在kubernetes上的device状态
	log.V(3).Info("update the device status")
	if err := r.Status().Update(ctx, d); err != nil {
		conditions.MarkFalse(d, devicev1alpha1.DeviceManagingCondition, "failed to update status of device on openyurt", clusterv1.ConditionSeverityWarning, err.Error())
		return err
	} else if len(failedPropertyNames) != 0 {
        // 没有成功修改的属性的名称
		err = fmt.Errorf("the following device properties failed to reconcile: %v", failedPropertyNames)
		conditions.MarkFalse(d, devicev1alpha1.DeviceManagingCondition, err.Error(), clusterv1.ConditionSeverityInfo, "")
		return err
	}
	conditions.MarkTrue(d, devicev1alpha1.DeviceManagingCondition)
	return nil
}

// Update the actual property value of the device on edge platform,
// return the latest status and the names of the property that failed to update
func (r *DeviceReconciler) reconcileDeviceProperties(d *devicev1alpha1.Device, deviceStatus *devicev1alpha1.DeviceStatus, log logr.Logger) (*devicev1alpha1.DeviceStatus, []string) {
	newDeviceStatus := deviceStatus.DeepCopy()
	// This list is used to hold the names of properties that failed to reconcile
    // 这个列表用来记录没有成功调整的属性的名称
	var failedPropertyNames []string
	// 2. reconciling the device properties' value
	log.V(3).Info("reconciling the device properties' value")
	for _, desiredProperty := range d.Spec.DeviceProperties {
		if desiredProperty.DesiredValue == "" {
			continue
		}
		propertyName := desiredProperty.Name
        // 获取属性对应的真实值
		// 1.1. gets the actual property value of the current device from edge platform
		log.V(4).Info("getting the actual property state", "property", propertyName)
		actualProperty, err := r.deviceCli.GetPropertyState(nil, propertyName, d, iotcli.GetOptions{})
		if err != nil {
            // 如果出错了，就将这个属性值放入列表中
			failedPropertyNames = append(failedPropertyNames, propertyName)
			continue
		}
		log.V(4).Info("got the actual property state",
			"property name", propertyName,
			"property getURL", actualProperty.GetURL,
			"property actual value", actualProperty.ActualValue)

		if newDeviceStatus.DeviceProperties == nil {
            // 如果是第一个属性，就要新建一个DeviceProperties
			newDeviceStatus.DeviceProperties = map[string]devicev1alpha1.ActualPropertyState{}
		}
        // 将相应的属性名和真实属性填入
		newDeviceStatus.DeviceProperties[propertyName] = *actualProperty

		// 1.2. set the device attribute in the edge platform to the expected value
		if desiredProperty.DesiredValue != actualProperty.ActualValue {
            // 如果想要的属性值与实际的不相同时
			log.V(4).Info("the desired value and the actual value are different",
				"desired value", desiredProperty.DesiredValue,
				"actual value", actualProperty.ActualValue)
            // 就将这个属性更新一下
			if err := r.deviceCli.UpdatePropertyState(nil, propertyName, d, iotcli.UpdateOptions{}); err != nil {
				log.V(4).Error(err, "failed to update property", "propertyName", propertyName)
                // 如果出错了，就将结果写入列表
				failedPropertyNames = append(failedPropertyNames, propertyName)
				continue
			}

			log.V(4).Info("successfully set the property to desired value", "property", propertyName)
            // 更新后的属性
			newActualProperty := devicev1alpha1.ActualPropertyState{
				Name:        propertyName,
				GetURL:      actualProperty.GetURL,
				ActualValue: desiredProperty.DesiredValue,
			}
			newDeviceStatus.DeviceProperties[propertyName] = newActualProperty
		}
	}
	return newDeviceStatus, failedPropertyNames
}

```

### device_syncer

```go

func (ds *DeviceSyncer) Run(stop <-chan struct{}) {
	ds.log.V(1).Info("starting the DeviceSyncer...")
	go func() {
		for {
            // 到了同步时间
			<-time.After(ds.syncPeriod)
			// 1. get device on edge platform and OpenYurt
            // 获取kubernetes上和edgex上的device列表
			edgeDevices, kubeDevices, err := ds.getAllDevices()
			if err != nil {
				ds.log.V(3).Error(err, "fail to list the devices")
				continue
			}

			// 2. find the device that need to be synchronized
            // 找到需要被同步的设备
			redundantEdgeDevices, redundantKubeDevices, syncedDevices := ds.findDiffDevice(edgeDevices, kubeDevices)
			ds.log.V(1).Info("The number of devices waiting for synchronization",
				"Edge device should be added to OpenYurt", len(redundantEdgeDevices),
				"OpenYurt device that should be deleted", len(redundantKubeDevices),
				"Devices that should be synchronized", len(syncedDevices))

			// 3. create device on OpenYurt which are exists in edge platform but not in OpenYurt
            // 在kubernetes平台上创建在edgex平台上但是不在kubernetes平台上的设备
			if err := ds.syncEdgeToKube(redundantEdgeDevices); err != nil {
				ds.log.V(3).Error(err, "fail to create devices on OpenYurt")
				continue
			}

			// 4. delete redundant device on OpenYurt
            // 删除kubernetes上多余的设备
			if err := ds.deleteDevices(redundantKubeDevices); err != nil {
				ds.log.V(3).Error(err, "fail to delete redundant devices on OpenYurt")
				continue
			}

			// 5. update device status on OpenYurt
            // 更新在kubernetes上的设备状态
			if err := ds.updateDevices(syncedDevices); err != nil {
				ds.log.Error(err, "fail to update devices status")
				continue
			}

			ds.log.V(1).Info("One round of Device synchronization is complete")

		}
	}()

	<-stop
	ds.log.V(1).Info("stopping the device syncer")
}

// Get the existing Device on the Edge platform, as well as OpenYurt existing Device
// edgeDevice：map[actualName]device
// kubeDevice：map[actualName]device
func (ds *DeviceSyncer) getAllDevices() (map[string]devicev1alpha1.Device, map[string]devicev1alpha1.Device, error) {
	edgeDevice := map[string]devicev1alpha1.Device{}
	kubeDevice := map[string]devicev1alpha1.Device{}
	// 1. list devices on edge platform
    // 获取在edgex平台上的设备列表
	eDevs, err := ds.deviceCli.List(nil, edgeCli.ListOptions{})
	if err != nil {
		ds.log.V(4).Error(err, "fail to list the devices object on the Edge Platform")
		return edgeDevice, kubeDevice, err
	}
	// 2. list devices on OpenYurt (filter objects belonging to edgeServer)
    // 获取在kubernetes平台上的设备列表
	var kDevs devicev1alpha1.DeviceList
	listOptions := client.MatchingFields{"spec.nodePool": ds.NodePool}
	if err = ds.List(context.TODO(), &kDevs, listOptions); err != nil {
		ds.log.V(4).Error(err, "fail to list the devices object on the OpenYurt")
		return edgeDevice, kubeDevice, err
	}
    // 将他们变成 map[actualName]device 这种样式
	for i := range eDevs {
		deviceName := getActualName(&eDevs[i])
		edgeDevice[deviceName] = eDevs[i]
	}

	for i := range kDevs.Items {
		deviceName := getActualName(&kDevs.Items[i])
		kubeDevice[deviceName] = kDevs.Items[i]
	}
	return edgeDevice, kubeDevice, nil
}

// Get the list of devices that need to be added, deleted and updated
func (ds *DeviceSyncer) findDiffDevice(
	edgeDevice map[string]devicev1alpha1.Device, kubeDevice map[string]devicev1alpha1.Device) (
	redundantEdgeDevices map[string]*devicev1alpha1.Device, redundantKubeDevices map[string]*devicev1alpha1.Device, syncedDevices map[string]*devicev1alpha1.Device) {
    
    // 获取需要被增加、删除、修改的设备列表

	redundantEdgeDevices = map[string]*devicev1alpha1.Device{}
	redundantKubeDevices = map[string]*devicev1alpha1.Device{}
	syncedDevices = map[string]*devicev1alpha1.Device{}

	for n := range edgeDevice {
        // 遍历edgex上设备的列表
		tmp := edgeDevice[n]
		edName := getActualName(&tmp)
		if _, exists := kubeDevice[edName]; !exists {
			// 如果在kubernetes上不存在这个内容
            ed := edgeDevice[n]
            // 那就放在添加的list里面
			redundantEdgeDevices[edName] = ds.completeCreateContent(&ed)
		} else {
            // 否则放在更新的list里面
			kd := kubeDevice[edName]
			ed := edgeDevice[edName]
			syncedDevices[edName] = ds.completeUpdateContent(&kd, &ed)
		}
	}

	for n, v := range kubeDevice {
        // 找到kubernetes上有，但是edgex上没有的设备
		if !v.Status.Synced {
			continue
		}
		tmp := kubeDevice[n]
		kdName := getActualName(&tmp)
		if _, exists := edgeDevice[kdName]; !exists {
			kd := kubeDevice[n]
			redundantKubeDevices[kdName] = &kd
		}
	}
	return
}

// syncEdgeToKube creates device on OpenYurt which are exists in edge platform but not in OpenYurt
func (ds *DeviceSyncer) syncEdgeToKube(edgeDevs map[string]*devicev1alpha1.Device) error {
	for _, ed := range edgeDevs {
        // 遍历这些设备，并在kubernetes上创建这些设备。
        // （这些设备是在edgex上有，但是在kubernetes行没有）
		if err := ds.Client.Create(context.TODO(), ed); err != nil {
			if apierrors.IsAlreadyExists(err) {
				continue
			}
			ds.log.V(5).Info("created device failed:",
				"device", strings.ToLower(ed.Name))
			return err
		}
	}
	return nil
}

// deleteDevices deletes redundant device on OpenYurt
func (ds *DeviceSyncer) deleteDevices(redundantKubeDevices map[string]*devicev1alpha1.Device) error {
    // 删除kubernetes上多余的设备
	for i := range redundantKubeDevices {
		if err := ds.Client.Delete(context.TODO(), redundantKubeDevices[i]); err != nil {
			ds.log.V(5).Error(err, "fail to delete the Device on OpenYurt",
				"device", redundantKubeDevices[i].Name)
			return err
		}
	}
	return nil
}

// updateDevicesStatus updates device status on OpenYurt
func (ds *DeviceSyncer) updateDevices(syncedDevices map[string]*devicev1alpha1.Device) error {
	// 对于两边都有的设备，进行更新
    for n := range syncedDevices {
		if err := ds.Client.Status().Update(context.TODO(), syncedDevices[n]); err != nil {
			if apierrors.IsConflict(err) {
				ds.log.Info("----Conflict")
				continue
			}
			return err
		}
	}
	return nil
}
// 下面两个函数是用在kubernetes和edgex两个平台的设备的互相转换的

// completeCreateContent completes the content of the device which will be created on OpenYurt
func (ds *DeviceSyncer) completeCreateContent(edgeDevice *devicev1alpha1.Device) *devicev1alpha1.Device {
	createDevice := edgeDevice.DeepCopy()
	createDevice.Spec.NodePool = ds.NodePool
	createDevice.Name = strings.Join([]string{ds.NodePool, createDevice.Name}, "-")
	createDevice.Spec.Managed = false

	return createDevice
}

// completeUpdateContent completes the content of the device which will be updated on OpenYurt
func (ds *DeviceSyncer) completeUpdateContent(kubeDevice *devicev1alpha1.Device, edgeDevice *devicev1alpha1.Device) *devicev1alpha1.Device {
	updatedDevice := kubeDevice.DeepCopy()
	_, aps, _ := ds.deviceCli.ListPropertiesState(nil, updatedDevice, edgeCli.ListOptions{})
	// update device status
	updatedDevice.Status.LastConnected = edgeDevice.Status.LastConnected
	updatedDevice.Status.LastReported = edgeDevice.Status.LastReported
	updatedDevice.Status.AdminState = edgeDevice.Status.AdminState
	updatedDevice.Status.OperatingState = edgeDevice.Status.OperatingState
	updatedDevice.Status.DeviceProperties = aps
	return updatedDevice
}

func getActualName(d *devicev1alpha1.Device) string {
	return d.Labels[EdgeXObjectName]
}
```















