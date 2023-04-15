## Interface

### manager

```go
package interfaces

type AutoEventManager interface {
	// StartAutoEvents starts all the AutoEvents of the device service
    // 这个函数用来启动device service的所有自动事件
	StartAutoEvents()
	// RestartForDevice restarts all the AutoEvents of the specific device
    // 这个函数用来重启指定设备的所有的自动事件
	RestartForDevice(name string)
	// StopForDevice stops all the AutoEvents of the specific device
    // 这个函数用来停止指定设备的所有的自动事件
	StopForDevice(name string)
}
```

### protocoldriver

```go
// ProtocolDriver is a low-level device-specific interface used by
// other components of an EdgeX Device Service to interact with
// a specific class of devices.
// ProtocolDriver是EdgeX设备服务的其他组件使用的特定于设备的低级接口，用于与特定类别的设备交互
type ProtocolDriver interface {
	// Initialize performs protocol-specific initialization for the device service.
	// The given *AsyncValues channel can be used to push asynchronous events and
	// readings to Core Data. The given []DiscoveredDevice channel is used to send
	// discovered devices that will be filtered and added to Core Metadata asynchronously.
    // Initialize为设备服务执行特定于协议的初始化。给定的*AsyncValues通道可用于将异步事件和读数推送到核心数据。给定的[]DiscoveredDevice通道用于发送发现的设备，这些设备将被异步筛选并添加到核心元数据中。
	Initialize(sdk DeviceServiceSDK) error

	// HandleReadCommands passes a slice of CommandRequest struct each representing
	// a ResourceOperation for a specific device resource.
    // HandleReadCommands传递一个CommandRequest结构的切片，
    // 每个切片代表一个特定设备资源的ResourceOperation。
	HandleReadCommands(deviceName string, protocols map[string]models.ProtocolProperties, reqs []sdkModels.CommandRequest) ([]*sdkModels.CommandValue, error)

	// HandleWriteCommands passes a slice of CommandRequest struct each representing
	// a ResourceOperation for a specific device resource.
	// Since the commands are actuation commands, params provide parameters for the individual
	// command.
    // HandleWriteCommands传递一个CommandRequest结构的切片，
    // 每个切片代表一个特定设备资源的ResourceOperation。
    // 由于这些命令是执行命令，由于这些命令是执行命令，params提供了各个命令的参数。
	HandleWriteCommands(deviceName string, protocols map[string]models.ProtocolProperties, reqs []sdkModels.CommandRequest, params []*sdkModels.CommandValue) error

	// Stop instructs the protocol-specific DS code to shutdown gracefully, or
	// if the force parameter is 'true', immediately. The driver is responsible
	// for closing any in-use channels, including the channel used to send async
	// readings (if supported).
    // 停止指示特定协议的DS代码优雅地关闭，或者如果强制参数为 "true"，则立即关闭。
    // 驱动程序负责关闭任何正在使用的通道，包括用于发送异步读数的通道（如果支持）。
	Stop(force bool) error

	// AddDevice is a callback function that is invoked
	// when a new Device associated with this Device Service is added
    // AddDevice是一个回调函数。当一个新的与该设备服务相关的设备被添加时被调用。
	AddDevice(deviceName string, protocols map[string]models.ProtocolProperties, adminState models.AdminState) error

	// UpdateDevice is a callback function that is invoked
	// when a Device associated with this Device Service is updated
    // UpdateDevice是一个回调函数。当一个新的与该设备服务相关的设备被更新时被调用。
	UpdateDevice(deviceName string, protocols map[string]models.ProtocolProperties, adminState models.AdminState) error

	// RemoveDevice is a callback function that is invoked
	// when a Device associated with this Device Service is removed
    // RemoveDevice是一个回调函数。当一个新的与该设备服务相关的设备被删除时被调用。
	RemoveDevice(deviceName string, protocols map[string]models.ProtocolProperties) error

	// Discover triggers protocol specific device discovery, asynchronously
	// writes the results to the channel which is passed to the implementation
	// via ProtocolDriver.Initialize(). The results may be added to the device service
	// based on a set of acceptance criteria (i.e. Provision Watchers).
    // Discover()触发特定协议的设备发现，异步进行将结果写入通道，
    // 通过ProtocolDriver.Initialize()传递给实现。
    // 这些结果可以根据一组接受标准（即供应观察者）添加到设备服务中。
	Discover() error

	// ValidateDevice triggers device's protocol properties validation, returns error
	// if validation failed and the incoming device will not be added into EdgeX.
    // ValidateDevice 触发设备的协议属性验证，如果验证失败返回错误。
    // 如果验证失败，传入的设备将不会被添加到EdgeX中。
	ValidateDevice(device models.Device) error
}
```

### service

```go
// DeviceServiceSDK defines the interface for an EdgeX Device Service SDK
// DeviceServiceSDK为EdgeX 设备服务SDK 提供了一个接口
type DeviceServiceSDK interface {
	// AddDevice adds a new Device to the Device Service and Core Metadata
	// Returns new Device id or non-nil error.
    // AddDevice给Device Service和Core Metadata增加一个设备
    // 返回一个新的设备或non-nil错误
	AddDevice(device models.Device) (string, error)
	// Devices return all managed Devices from cache
    // Devices从缓存中返回所有被管理的设备
	Devices() []models.Device
	// GetDeviceByName returns the Device by its name if it exists in the cache, or returns an error.
    // GetDeviceByName 通过设备名称返回设备（如果他在cache中的话，或者返回一个错误）
	GetDeviceByName(name string) (models.Device, error)
	// UpdateDevice updates the Device in the cache and ensures that the
	// copy in Core Metadata is also updated.
    // UpdateDevice 更新在cache中的Device并且确保在Core Metadata中的副本也被更新了
	UpdateDevice(device models.Device) error
	// RemoveDeviceByName removes the specified Device by name from the cache and ensures that the
	// instance in Core Metadata is also removed.
    // RemoveDeviceByName通过名字从cache中移除特定的设备，并且确保在Core Metadta中的实例也被移除了。
	RemoveDeviceByName(name string) error
	// AddDeviceProfile adds a new DeviceProfile to the Device Service and Core Metadata
	// Returns new DeviceProfile id or non-nil error.
    // AddDeviceProfile给Device Service和Core Metadata增加一个Device Service。返回新的DeviceProfile id或者非空错误
	AddDeviceProfile(profile models.DeviceProfile) (string, error)
	// DeviceProfiles return all managed DeviceProfiles from cache
    // DeviceProfiles返回所有cache管理的DeviceProfiles
	DeviceProfiles() []models.DeviceProfile
	// GetProfileByName returns the Profile by its name if it exists in the cache, or returns an error.
    // GetProfileByName返回通过名称找到的Profile，如果在cache中存在的话，或者返回一个错误
	GetProfileByName(name string) (models.DeviceProfile, error)
	// UpdateDeviceProfile updates the DeviceProfile in the cache and ensures that the
	// copy in Core Metadata is also updated.
    // UpdateDeviceProfile更新在cache中的DeviceProfile并且确保在Core Metadata中的副本也被更新
	UpdateDeviceProfile(profile models.DeviceProfile) error
	// RemoveDeviceProfileByName removes the specified DeviceProfile by name from the cache and ensures that the
	// instance in Core Metadata is also removed.
    // RemoveDeviceProfileByName根据名称从cache中移除特定的DeviceProfile，并且确保在Core Metadata中的实例也被移除了
	RemoveDeviceProfileByName(name string) error
	// AddProvisionWatcher adds a new Watcher to the cache and Core Metadata
	// Returns new Watcher id or non-nil error.
    // AddProvisionWatcher给cache和Core Metadata增加了一个新的Watcher。
    // 返回一个新的Watcher id或者非空错误。
	AddProvisionWatcher(watcher models.ProvisionWatcher) (string, error)
	// ProvisionWatchers return all managed Watchers from cache
    // ProvisionWatchers返回所有由cache管理的Watchers
	ProvisionWatchers() []models.ProvisionWatcher
	// GetProvisionWatcherByName returns the Watcher by its name if it exists in the cache, or returns an error.
    // GetProvisionWatcherByName通过名字返回Watcher，如果他存在在cache，否则返回一个错误。
	GetProvisionWatcherByName(name string) (models.ProvisionWatcher, error)
	// UpdateProvisionWatcher updates the Watcher in the cache and ensures that the
	// copy in Core Metadata is also updated.
    // UpdateProvisionWatcher更新在cache中的Watcher并且确保在Core Metadata中的副本也被更新了。
	UpdateProvisionWatcher(watcher models.ProvisionWatcher) error
	// RemoveProvisionWatcher removes the specified Watcher by name from the cache and ensures that the
	// instance in Core Metadata is also removed.
    // RemoveProvisionWatcher通过名称在cache移除特定的Watcher，并且确保在Core Metadata中的实例也被移除了。
	RemoveProvisionWatcher(name string) error
	// DeviceResource retrieves the specific DeviceResource instance from cache according to
	// the Device name and Device Resource name
    // DeviceResource从cache中根据Device name和Device Resource name获取特定的DeviceResource实例
	DeviceResource(deviceName string, deviceResource string) (models.DeviceResource, bool)
	// DeviceCommand retrieves the specific DeviceCommand instance from cache according to
	// the Device name and Command name
    // DeviceCommand从cache中根据Device name和DeviceCommand name获取特定的DeviceCommand实例
	DeviceCommand(deviceName string, commandName string) (models.DeviceCommand, bool)
	// AddDeviceAutoEvent adds a new AutoEvent to the Device with given name
    // AddDeviceAutoEvent给指定名称的设备添加一个新的AutoEvent
	AddDeviceAutoEvent(deviceName string, event models.AutoEvent) error
	// RemoveDeviceAutoEvent removes an AutoEvent from the Device with given name
    // RemoveDeviceAutoEvent给指定名称的设备移除一个AutoEvent
	RemoveDeviceAutoEvent(deviceName string, event models.AutoEvent) error
	// SetDeviceOpState sets the operating state of device
    // SetDeviceOpState 设置设备的operating state
	SetDeviceOpState(name string, state models.OperatingState) error
	// UpdateDeviceOperatingState updates the Device's OperatingState with given name
	// in Core Metadata and device service cache.
	// UpdateDeviceOperatingState 更新在Core Metadata和cache中给定设备名称的设备的OperatingState
    UpdateDeviceOperatingState(deviceName string, state string) error

	// Name returns the name of this Device Service
    // 返回这个Device Service的名称
	Name() string

	// Version returns the version number of this Device Service
    // 返回这个Device Service的版本号
	Version() string

	// AsyncReadingsEnabled returns a bool value to indicate whether the asynchronous reading is enabled.
    // AsyncReadingsEnabled 返回一个布尔值表明是否异步读取是开启的
	AsyncReadingsEnabled() bool

	// AsyncValuesChannel returns a channel to allow developer send asynchronous reading back to SDK.
    // AsyncValuesChannel 返回一个通道允许开发者发送异步读取到SDK中
	AsyncValuesChannel() chan *sdkModels.AsyncValues

	// DiscoveredDeviceChannel returns a channel to allow developer send discovered devices back to SDK.
    // DiscoverrdDeviceChannel 返回一个通道用来允许开发者发送发现的设备给SDK
	DiscoveredDeviceChannel() chan []sdkModels.DiscoveredDevice

	// DeviceDiscoveryEnabled returns a bool value to indicate whether device discovery is enabled.
    // DeviceDiscoveryEnabled 返回一个布尔值来表明是否设备发现是允许的
	DeviceDiscoveryEnabled() bool

	// DriverConfigs retrieves the driver specific configuration
    // DriverConfigs 或者驱动特定的配置
	DriverConfigs() map[string]string

	// AddRoute allows leveraging the existing internal web server to add routes specific to Device Service.
    // AddRoute允许利用现有的内部网络服务器来添加特定于设备服务的路由。
	AddRoute(route string, handler func(http.ResponseWriter, *http.Request), methods ...string) error

	// LoadCustomConfig uses the Config Processor from go-mod-bootstrap to attempt to load service's
	// custom configuration. It uses the same command line flags to process the custom config in the same manner
	// as the standard configuration.
    // LoadCustomConfig使用go-mod-bootstrap的配置处理器来尝试加载服务的自定义配置。它使用相同的命令行标志，以与标准配置相同的方式处理自定义配置。
	LoadCustomConfig(customConfig UpdatableConfig, sectionName string) error

	// ListenForCustomConfigChanges uses the Config Processor from go-mod-bootstrap to attempt to listen for
	// changes to the specified custom configuration section. LoadCustomConfig must be called previously so that
	// the instance of sdkService.configProcessor has already been set.
    // ListenForCustomConfigChanges使用go-mod-bootstrap的配置处理器，试图监听指定的自定义配置部分的变化。必须事先调用LoadCustomConfig，以便 sdkService.configProcessor的实例已经被设置。
	ListenForCustomConfigChanges(configToWatch interface{}, sectionName string, changedCallback func(interface{})) error

	// LoggingClient returns the logger.LoggingClient.
    // LoggingClient 返回logger.LoggingClient
	LoggingClient() logger.LoggingClient

	// SecretProvider returns the interfaces.SecretProvider.
    // SecretProvider 返回interfaces.SecretProvider
	SecretProvider() interfaces.SecretProvider

	// MetricsManager returns the Metrics Manager used to register counter, gauge, gaugeFloat64 or timer metric types from
	// github.com/rcrowley/go-metrics
    // 指标管理器返回用于从 github.com/rcrowley/go-metrics 注册计数器、仪表、仪表、仪表 Float64 或计时器指标类型的指标管理器
	MetricsManager() interfaces.MetricsManager
}
```

## Models

### asyncvalues

```go
// AsyncValues is the struct for sending Device readings asynchronously via ProtocolDrivers
// AsyncValues 是用来通过ProtocolDrivers异步发送设备读请求的结构体
type AsyncValues struct {
	DeviceName    string
	SourceName    string
	CommandValues []*CommandValue
}
```

### commandrequest

```go
// CommandRequest is the struct for requesting a command to ProtocolDrivers
// CommandRequest 是用于向ProtocolDrivers请求命令的结构
type CommandRequest struct {
	// DeviceResourceName is the name of Device Resource for this command
    // DeviceResourceName 是这条命令请求的设备资源名称
	DeviceResourceName string
	// Attributes is a key/value map to represent the attributes of the Device Resource
    // Attributes 是一个key/value的映射来表示设备资源的属性
	Attributes map[string]interface{}
	// Type is the data type of the Device Resource
    // Type是设备资源的数据类型
	Type string
}
```

### commandvalue

```go
// CommandValue is the struct to represent the reading value of a Get command coming
// from ProtocolDrivers or the parameter of a Put command sending to ProtocolDrivers.
// CommandValue是表示即将发布的 Get 命令的读取值的结构从ProtocolDrivers或
// 发送到ProtocolDrivers的 Put 命令的参数。
type CommandValue struct {
	// DeviceResourceName is the name of Device Resource for this command
    // DeviceResourceName 是发给这个命令的Device Resource的名称
	DeviceResourceName string
	// Type indicates what type of value was returned from the ProtocolDriver instance in
	// response to HandleCommand being called to handle a single ResourceOperation.
    // Type 表明了指示从 ProtocolDriver 实例返回的值类型，以响应为处理单个资源操作而调用的 HandleCommand。
	Type string
	// Value holds value returned by a ProtocolDriver instance.
	// The value can be converted to its native type by referring to ValueType.
    // Value 保存由 ProtocolDriver 实例返回的值。
	// 可以通过引用 ValueType 将该值转换为其原本类型。
	Value interface{}
	// Origin is an int64 value which indicates the time the reading
	// contained in the CommandValue was read by the ProtocolDriver
	// instance.
    // Origin是一个 int64 值，表示读数的时间包含在命令值中，由ProtocolDriver实例读取。
	Origin int64
	// Tags allows device service to add custom information to the Event in order to
	// help identify its origin or otherwise label it before it is send to north side.
    // Tags允许设备服务向事件添加自定义信息，以便在发送到北侧之前，帮助识别其来源或以其他方式标记它。
	Tags map[string]string
}

// NewCommandValue create a CommandValue according to the valueType supplied.
func NewCommandValue(deviceResourceName string, valueType string, value interface{}) (*CommandValue, error) {
	err := validate(valueType, value)
	if err != nil {
		return nil, errors.NewCommonEdgeX(errors.KindServerError, "failed to create CommandValue", err)
	}

	return &CommandValue{
		DeviceResourceName: deviceResourceName,
		Type:               valueType,
		Value:              value,
		Tags:               make(map[string]string)}, nil
}

// NewCommandValueWithOrigin wraps NewCommandValue, create a CommandValue and add the Origin field.
func NewCommandValueWithOrigin(deviceResourceName string, valueType string, value interface{}, origin int64) (*CommandValue, error) {
	cv, err := NewCommandValue(deviceResourceName, valueType, value)
	if err != nil {
		return nil, errors.NewCommonEdgeXWrapper(err)
	}

	cv.Origin = origin
	return cv, nil
}
// 下面是一些有关type的操作了。
```

### discovereddevice

```go
// DiscoveredDevice defines the required information for a found device.
// DiscoveredDevice定义了找到一个设备的必要信息
type DiscoveredDevice struct {
	Name        string
	Protocols   map[string]models.ProtocolProperties
	Description string
	Labels      []string
}
```

## Service

```go
const EnvInstanceName = "EDGEX_INSTANCE_NAME"

type deviceService struct {
    // 这个是service名称
	serviceKey         string
    // 这个用来写日志
	lc                 logger.LoggingClient
    // 这个是驱动接口，主要实现与设备发命令、对设备的增删改操作
	driver             interfaces.ProtocolDriver
    // 这个是自动事件驱动，主要实现给设备增加、删除、停止自动事件
	autoEventManager   interfaces.AutoEventManager
    // 这个是rest请求的controller
	controller         *restController.RestController
    // 这个是异步信号，发送的是一个通道，里面的值类型是AsyncValues
	asyncCh            chan *sdkModels.AsyncValues
    // 这个是设备控制信号，发送的是一个通道切片，里面的值类型是DiscoveredDevice
	deviceCh           chan []sdkModels.DiscoveredDevice
    // 使用默认标志
	flags              *flags.Default
    // 这里放DeviceService实体
	deviceServiceModel *models.DeviceService
    // device service 的配置
	config             *config.ConfigurationStruct
    // 这个是配置的处理
	configProcessor    *bootstrapConfig.Processor
	wg                 *sync.WaitGroup
	ctx                context.Context
	dic                *di.Container
}

func NewDeviceService(serviceKey string, serviceVersion string, driver interfaces.ProtocolDriver) (*deviceService, error) {
	var service deviceService
    // 校验serviceKey的内容，如果为空，就报错
	if serviceKey == "" {
		return nil, errors.New("please specify device service name")
	}
    // 配置serviceKey
	service.serviceKey = serviceKey
	
    // 校验serviceVersion的内容，如果为空，就报错
	if serviceVersion == "" {
		return nil, errors.New("please specify device service version")
	}
    // 配置ServiceVersion
	sdkCommon.ServiceVersion = serviceVersion

    // 加入驱动
	service.driver = driver

	service.config = &config.ConfigurationStruct{}
    // 返回这个service
	return &service, nil
}

func (s *deviceService) Run() error {
	var instanceName string
	startupTimer := startup.NewStartUpTimer(s.serviceKey)

	additionalUsage :=
		"    -i, --instance                  Provides a service name suffix which allows unique instance to be created\n" +
			"                                    If the option is provided, service name will be replaced with \"<name>_<instance>\"\n"
	s.flags = flags.NewWithUsage(additionalUsage)
	s.flags.FlagSet.StringVar(&instanceName, "instance", "", "")
	s.flags.FlagSet.StringVar(&instanceName, "i", "", "")
	s.flags.Parse(os.Args[1:])
    // 加载配置，设置service名称 serviceKey + instanceName
	s.setServiceName(instanceName)
    
	// 新建配置，这个配置后面可能会被反序列化，可能会从文件里读出来
	s.config = &config.ConfigurationStruct{}
    // 新建device service
	s.deviceServiceModel = &models.DeviceService{Name: s.serviceKey}
	
    // 这里是在干嘛？填充dic？
	s.dic = di.NewContainer(di.ServiceConstructorMap{
		container.ConfigurationName: func(get di.Get) interface{} {
			return s.config
		},
		container.DeviceServiceName: func(get di.Get) interface{} {
			return s.deviceServiceModel
		},
		container.ProtocolDriverName: func(get di.Get) interface{} {
			return s.driver
		},
	})
	
    // 新建一个http（restful）请求的router
	router := mux.NewRouter()
    // 新建一个httpServer
	httpServer := handlers.NewHttpServer(router, true)

	ctx, cancel := context.WithCancel(context.Background())
	wg, deferred, successful := bootstrap.RunAndReturnWaitGroup(
		ctx,
		cancel,
		s.flags,
		s.serviceKey,
		common.ConfigStemDevice,
		s.config,
		nil,
		startupTimer,
		s.dic,
		true,
		bootstrapTypes.ServiceTypeDevice,
        // 上面都是些配置信息，其实不重要，重要的是下面的这些handler
        // 一个一个来看这些handler
		[]bootstrapInterfaces.BootstrapHandler{
			httpServer.BootstrapHandler,
			messageBusBootstrapHandler,
			handlers.NewServiceMetrics(s.serviceKey).BootstrapHandler, // Must be after Messaging
			handlers.NewClientsBootstrap().BootstrapHandler,
			autoevent.BootstrapHandler,
			NewBootstrap(s, router).BootstrapHandler,
			autodiscovery.BootstrapHandler,
			handlers.NewStartMessage(s.serviceKey, sdkCommon.ServiceVersion).BootstrapHandler,
		})

	defer func() {
		deferred()
		s.Stop(false)
	}()

	if !successful {
		cancel()
		return errors.New("bootstrapping failed")
	}

	// TODO: call ProtocolDriver.Start() proposed in issue#1339

	wg.Wait()
	return nil
}

// Name returns the name of this Device Service
func (s *deviceService) Name() string {
    // 返回这个Device Service的名称
	return s.serviceKey
}

// Version returns the version number of this Device Service
func (s *deviceService) Version() string {
    // 返回这个Device Service的版本号
	return sdkCommon.ServiceVersion
}

// SecretProvider returns the SecretProvider
func (s *deviceService) SecretProvider() bootstrapInterfaces.SecretProvider {
	return bootstrapContainer.SecretProviderFrom(s.dic.Get)
}

// MetricsManager returns the Metrics Manager used to register counter, gauge, gaugeFloat64 or timer metric types from
// github.com/rcrowley/go-metrics
func (s *deviceService) MetricsManager() bootstrapInterfaces.MetricsManager {
	return bootstrapContainer.MetricsManagerFrom(s.dic.Get)
}

// LoggingClient returns the logger.LoggingClient
func (s *deviceService) LoggingClient() logger.LoggingClient {
	return s.lc
}

// AsyncReadings returns a bool value to indicate whether the asynchronous reading is enabled.
func (s *deviceService) AsyncReadingsEnabled() bool {
	return s.config.Device.EnableAsyncReadings
}

func (s *deviceService) AsyncValuesChannel() chan *sdkModels.AsyncValues {
	return s.asyncCh
}

// DeviceDiscovery returns a bool value to indicate whether the device discovery is enabled.
func (s *deviceService) DeviceDiscoveryEnabled() bool {
	return s.config.Device.Discovery.Enabled
}

func (s *deviceService) DiscoveredDeviceChannel() chan []sdkModels.DiscoveredDevice {
	return s.deviceCh
}

// AddRoute allows leveraging the existing internal web server to add routes specific to Device Service.
func (s *deviceService) AddRoute(route string, handler func(http.ResponseWriter, *http.Request), methods ...string) error {
	return s.controller.AddRoute(route, handler, methods...)
}

// LoadCustomConfig uses the Config Processor from go-mod-bootstrap to attempt to load service's
// custom configuration. It uses the same command line flags to process the custom config in the same manner
// as the standard configuration.
func (s *deviceService) LoadCustomConfig(customConfig interfaces.UpdatableConfig, sectionName string) error {
	if s.configProcessor == nil {
		s.configProcessor = bootstrapConfig.NewProcessorForCustomConfig(s.flags, s.ctx, s.wg, s.dic)
	}

	if err := s.configProcessor.LoadCustomConfigSection(customConfig, sectionName); err != nil {
		return err
	}

	s.controller.SetCustomConfigInfo(customConfig)

	return nil
}

// ListenForCustomConfigChanges uses the Config Processor from go-mod-bootstrap to attempt to listen for
// changes to the specified custom configuration section. LoadCustomConfig must be called previously so that
// the instance of svc.configProcessor has already been set.
func (s *deviceService) ListenForCustomConfigChanges(
	configToWatch interface{},
	sectionName string,
	changedCallback func(interface{})) error {
	if s.configProcessor == nil {
		return fmt.Errorf(
			"custom configuration must be loaded before '%s' section can be watched for changes",
			sectionName)
	}

	s.configProcessor.ListenForCustomConfigChanges(configToWatch, sectionName, changedCallback)
	return nil
}

// selfRegister register device service itself onto metadata.
func (s *deviceService) selfRegister() edgexErr.EdgeX {
	localDeviceService := models.DeviceService{
		Name:        s.serviceKey,
		Labels:      s.config.Device.Labels,
		BaseAddress: bootstrapTypes.DefaultHttpProtocol + "://" + s.config.Service.Host + ":" + strconv.FormatInt(int64(s.config.Service.Port), 10),
		AdminState:  models.Unlocked,
	}
	*s.deviceServiceModel = localDeviceService
	ctx := context.WithValue(context.Background(), common.CorrelationHeader, uuid.NewString()) // nolint:staticcheck
	dsc := bootstrapContainer.DeviceServiceClientFrom(s.dic.Get)

	s.lc.Debugf("trying to find device service %s", localDeviceService.Name)
	res, err := dsc.DeviceServiceByName(ctx, localDeviceService.Name)
	if err != nil {
		if edgexErr.Kind(err) == edgexErr.KindEntityDoesNotExist {
			s.lc.Infof("device service %s doesn't exist, creating a new one", localDeviceService.Name)
			req := requests.NewAddDeviceServiceRequest(dtos.FromDeviceServiceModelToDTO(localDeviceService))
			idRes, err := dsc.Add(ctx, []requests.AddDeviceServiceRequest{req})
			if err != nil {
				s.lc.Errorf("failed to add device service %s: %v", localDeviceService.Name, err)
				return err
			}
			s.deviceServiceModel.Id = idRes[0].Id
			s.lc.Debugf("new device service id: %s", localDeviceService.Id)
		} else {
			s.lc.Errorf("failed to find device service %s", localDeviceService.Name)
			return err
		}
	} else {
		s.lc.Infof("device service %s exists, updating it", s.serviceKey)
		req := requests.NewUpdateDeviceServiceRequest(dtos.FromDeviceServiceModelToUpdateDTO(localDeviceService))
		req.Service.Id = nil
		_, err = dsc.Update(ctx, []requests.UpdateDeviceServiceRequest{req})
		if err != nil {
			s.lc.Errorf("failed to update device service %s with local config: %v", localDeviceService.Name, err)
			oldDeviceService := dtos.ToDeviceServiceModel(res.Service)
			*s.deviceServiceModel = oldDeviceService
		}
	}

	return nil
}

// DriverConfigs retrieves the driver specific configuration
func (s *deviceService) DriverConfigs() map[string]string {
	return s.config.Driver
}

// Stop shuts down the Service
func (s *deviceService) Stop(force bool) {
	// 强行停止这个service
    err := s.driver.Stop(force)
	if err != nil {
		s.lc.Errorf(err.Error())
	}
}

func (s *deviceService) setServiceName(instanceName string) {
	envValue := os.Getenv(EnvInstanceName)
    // 如果有环境名称，那么就加环境名称，否则直接发instanceName
	if len(envValue) > 0 {
		instanceName = envValue
	}

	if len(instanceName) > 0 {
		s.serviceKey = s.serviceKey + "_" + instanceName
	}
}
```

### httpServer.BootstrapHandler

> listenAndServe，主要是注册TimeoutHandler、RequestLimitMiddleware、ProcessCORS handlers，这些都是比较通用的handler。

```go
// BootstrapHandler fulfills the BootstrapHandler contract.  It creates two go routines -- one that executes ListenAndServe()
// and another that waits on closure of a context's done channel before calling Shutdown() to cleanly shut down the
// http server.
// BootstrapHandler 履行 BootstrapHandler 合约。它创建了两个go的协程，一个用来执行ListenAndServe()另一个等待上下文的 done 通道关闭，然后调用 Shutdown（） 以干净地关闭HTTP 服务器。
func (b *HttpServer) BootstrapHandler(
	ctx context.Context,
	wg *sync.WaitGroup,
	_ startup.Timer,
	dic *di.Container) bool {

	lc := container.LoggingClientFrom(dic.Get)
	
    // 这里将isRunning设置为true
	if !b.doListenAndServe {
		lc.Info("Web server intentionally NOT started.")
		wg.Add(1)
		go func() {
			defer wg.Done()

			b.isRunning = true
			<-ctx.Done()
			b.isRunning = false
		}()
		return true

	}

    // 获取bootstrapConfig
	bootstrapConfig := container.ConfigurationFrom(dic.Get).GetBootstrap()

	// this allows env override to explicitly set the value used
	// for ListenAndServe as needed for different deployments
    // 这允许 env 覆盖显式设置使用的值，对于不同部署的需要，用于ListenAndServe
	port := strconv.Itoa(bootstrapConfig.Service.Port)
	addr := bootstrapConfig.Service.ServerBindAddr + ":" + port
	// for backwards compatibility, the Host value is the default value if
	// the ServerBindAddr value is not specified
    // 为了向后兼容，如果未指定 ServerBindAddr 值，则主机值为默认值
	if bootstrapConfig.Service.ServerBindAddr == "" {
		addr = bootstrapConfig.Service.Host + ":" + port
	}
	
    // timeout是配置中的请求超时时间
	timeout, err := time.ParseDuration(bootstrapConfig.Service.RequestTimeout)
	if err != nil {
		lc.Errorf("unable to parse RequestTimeout value of %s to a duration: %v", bootstrapConfig.Service.RequestTimeout, err)
		return false
	}
	// 设置超时handler
	b.router.Use(func(next http.Handler) http.Handler {
		return http.TimeoutHandler(next, timeout, "HTTP request timeout")
	})
	// 设置最大请求大小handler
	b.router.Use(RequestLimitMiddleware(bootstrapConfig.Service.MaxRequestSize, lc))
	// 设置CORS handler
	b.router.Use(ProcessCORS(bootstrapConfig.Service.CORSConfiguration))

	// handle the CORS preflight request
    // 处理CORS提前请求
	b.router.Methods(http.MethodOptions).MatcherFunc(func(r *http.Request, rm *mux.RouteMatch) bool {
		return r.Header.Get(AccessControlRequestMethod) != ""
	}).HandlerFunc(HandlePreflight(bootstrapConfig.Service.CORSConfiguration))

    // 配置server地址
	server := &http.Server{
		Addr:              addr,
		Handler:           b.router,
		ReadHeaderTimeout: 5 * time.Second, // G112: A configured ReadHeaderTimeout in the http.Server averts a potential Slowloris Attack
	}

	wg.Add(1)
	go func() {
		defer wg.Done()

		<-ctx.Done()
		_ = server.Shutdown(context.Background())
		lc.Info("Web server shut down")
	}()

	lc.Info("Web server starting (" + addr + ")")
	
    // 执行ListenAndServer
	wg.Add(1)
	go func() {
		defer func() {
			wg.Done()
			b.isRunning = false
		}()
		
		b.isRunning = true
		err := server.ListenAndServe()
		// "Server closed" error occurs when Shutdown above is called in the Done processing, so it can be ignored
		if err != nil && err != http.ErrServerClosed {
			// Other errors occur during bootstrapping, like port bind fails, are considered fatal
			lc.Errorf("Web server failed: %v", err)

			// Allow any long-running go functions that may have started to stop before exiting
			cancel := container.CancelFuncFrom(dic.Get)
			cancel()

			// Wait for all long-running go functions to stop before exiting.
			wg.Done() // Must do this to account for this go func's wg.Add above otherwise wait will block indefinitely
			wg.Wait()
			os.Exit(1)
		} else {
			lc.Info("Web server stopped")
		}
	}()

	return true
}
```

### messageBusBootstrapHandler

> 根据配置中的MessageQueue配置，创建messageClient并且更新到di中，其中分别有publish、subscribe信息。对于EdgeX来说，默认的MessageBus实现是Redis，其中发布和订阅分别是通过redis的publish,psubscribe来实现的。

```go
func messageBusBootstrapHandler(ctx context.Context, wg *sync.WaitGroup, startupTimer startup.Timer, dic *di.Container) bool {
    // 这里是在构建messageBus客户端
	if !handlers.MessagingBootstrapHandler(ctx, wg, startupTimer, dic) {
		return false
	}

	lc := bootstrapContainer.LoggingClientFrom(dic.Get)
    // 这里进行订阅，异步处理command请求
	err := messaging.SubscribeCommands(ctx, dic)
	if err != nil {
		lc.Errorf("Failed to subscribe internal command request: %v", err)
		return false
	}

	err = messaging.MetadataSystemEventsCallback(ctx, dic)
	if err != nil {
		lc.Errorf("Failed to subscribe Metadata system events: %v", err)
		return false
	}

	err = messaging.SubscribeDeviceValidation(ctx, dic)
	if err != nil {
		lc.Errorf("Failed to subscribe device validation request: %v", err)
		return false
	}

	return true
}
```

#### MessagingBootstrapHandler

```go
// MessagingBootstrapHandler fulfills the BootstrapHandler contract.  If creates and initializes the Messaging client
// and adds it to the DIC
// MessagingBootstrapHandler 履行 BootstrapHandler 合约。
// 如果创建并初始化消息传递客户端并将其添加到 DIC
func MessagingBootstrapHandler(ctx context.Context, wg *sync.WaitGroup, startupTimer startup.Timer, dic *di.Container) bool {
	lc := container.LoggingClientFrom(dic.Get)
    // 来获取整个的配置信息
	configuration := container.ConfigurationFrom(dic.Get)

	messageBus := configuration.GetBootstrap().MessageBus
    // 如果messageBus没有启用
	if messageBus.Disabled {
        // 打印信息后，返回true
		lc.Info("MessageBus is disabled in configuration, skipping setup.")
		return true
	}
	// 如果messageBus没有填写Host的话
	if len(messageBus.Host) == 0 {
		lc.Error("MessageBus configuration not set or missing from service's GetBootstrap() implementation")
        // 这里return false？
		return false
	}

	// Make sure the MessageBus password is not leaked into the Service Config that can be retrieved via the /config endpoint
    // 确保 MessageBus 密码不会泄漏到可通过 /config 终结点检索的服务配置中
	messageBusInfo := deepCopy(messageBus)

	if len(messageBusInfo.AuthMode) > 0 &&
		!strings.EqualFold(strings.TrimSpace(messageBusInfo.AuthMode), boostrapMessaging.AuthModeNone) {
        // 获取messageBus的配置？
		if err := boostrapMessaging.SetOptionsAuthData(&messageBusInfo, lc, dic); err != nil {
			lc.Errorf("setting the MessageBus auth options failed: %v", err)
			return false
		}
	}
	// 这里就是在配置messageBus的信息
	msgClient, err := messaging.NewMessageClient(
		types.MessageBusConfig{
			Broker: types.HostInfo{
				Host:     messageBusInfo.Host,
				Port:     messageBusInfo.Port,
				Protocol: messageBusInfo.Protocol,
			},
			Type:     messageBusInfo.Type,
			Optional: messageBusInfo.Optional,
		})

	if err != nil {
		lc.Errorf("Failed to create MessageClient: %v", err)
		return false
	}
	// HasNotElapsed 返回构造期间指定的持续时间是否已过
	for startupTimer.HasNotElapsed() {
		select {
		case <-ctx.Done():
			return false
		default:
			err = msgClient.Connect()
			if err != nil {
				lc.Warnf("Unable to connect MessageBus: %s", err.Error())
				startupTimer.SleepForInterval()
				continue
			}

			wg.Add(1)
			go func() {
				defer wg.Done()
				<-ctx.Done()
                // 结束了的话
				if msgClient != nil {
					_ = msgClient.Disconnect()
				}
				lc.Infof("Disconnected from MessageBus")
			}()
			// 更新MessageClient？？
			dic.Update(di.ServiceConstructorMap{
				container.MessagingClientName: func(get di.Get) interface{} {
					return msgClient
				},
			})
			// 链接到这个MessageBus
			lc.Infof(
				"Connected to %s Message Bus @ %s://%s:%d with AuthMode='%s'",
				messageBusInfo.Type,
				messageBusInfo.Protocol,
				messageBusInfo.Host,
				messageBusInfo.Port,
				messageBusInfo.AuthMode)

			return true
		}
	}

	lc.Error("Connecting to MessageBus time out")
	return false
}
```

#### SubscribeCommands

```go
func SubscribeCommands(ctx context.Context, dic *di.Container) errors.EdgeX {
	lc := bootstrapContainer.LoggingClientFrom(dic.Get)
    // 从Container中获取messageBus的信息
	messageBusInfo := container.ConfigurationFrom(dic.Get).MessageBus
    // 获取deviceService
	deviceService := container.DeviceServiceFrom(dic.Get)
	
    // 构造需要订阅的topic
	requestSubscribeTopic := common.BuildTopic(messageBusInfo.GetBaseTopicPrefix(), common.CommandRequestSubscribeTopic, deviceService.Name, "#")
	lc.Infof("Subscribing to command requests on topic: %s", requestSubscribeTopic)
	
    // 构造响应topic的前缀
	responsePublishTopicPrefix := common.BuildTopic(messageBusInfo.GetBaseTopicPrefix(), common.ResponseTopic, deviceService.Name)
	lc.Infof("Responses to command requests will be published on topic: %s/<requestId>", responsePublishTopicPrefix)

	messages := make(chan types.MessageEnvelope)
	messageErrors := make(chan error)
	topics := []types.TopicChannel{
		{
			Topic:    requestSubscribeTopic,
			Messages: messages,
		},
	}
	// 获取设置好的messageBus
	messageBus := bootstrapContainer.MessagingClientFrom(dic.Get)
    // 对请求的topic发起订阅
	err := messageBus.Subscribe(topics, messageErrors)
	if err != nil {
		return errors.NewCommonEdgeXWrapper(err)
	}

	go func() {
        // 开启协程，来接收订阅topic发送的内容
		for {
			select {
			case <-ctx.Done():
				lc.Infof("Exiting waiting for MessageBus '%s' topic messages", requestSubscribeTopic)
				return
			case err = <-messageErrors:
				lc.Error(err.Error())
			case msgEnvelope := <-messages:
                // 收到消息，放入msgEnvelope
				lc.Debugf("Command request received on message queue. Topic: %s, Correlation-id: %s", msgEnvelope.ReceivedTopic, msgEnvelope.CorrelationID)

				// expected command request topic scheme: #/<service-name>/<device-name>/<command-name>/<method>
                // 期待命令请求topic的模式为：#/<service-name>/<device-name>/<command-name>/<method>
                // 根据 / 拆分收到的topic
				topicLevels := strings.Split(msgEnvelope.ReceivedTopic, "/")
				length := len(topicLevels)
				if length < 4 {
					lc.Error("Failed to parse and construct command response topic scheme, expected request topic scheme: '#/<service-name>/<device-name>/<command-name>/<method>'")
					continue
				}

				// expected command response topic scheme: #/<service-name>/<device-name>/<command-name>/<method>
				deviceName := topicLevels[length-3]
				commandName, err := url.QueryUnescape(topicLevels[length-2])
				if err != nil {
					lc.Errorf("Failed to unescape command name '%s'", commandName)
					continue
				}
				method := topicLevels[length-1]
				// 这里构造出响应topic
				responsePublishTopic := common.BuildTopic(responsePublishTopicPrefix, msgEnvelope.RequestID)

				switch strings.ToUpper(method) {
                // 如果是GET命令，就发起getCommand
				case "GET":
					getCommand(ctx, msgEnvelope, responsePublishTopic, deviceName, commandName, dic)
                // 如果是SET命令，就发起setCommand
				case "SET":
					setCommand(ctx, msgEnvelope, responsePublishTopic, deviceName, commandName, dic)
				default:
					lc.Errorf("unknown command method '%s', only 'get' or 'set' is allowed", method)
					continue
				}

				lc.Debugf("Command response published on message queue. Topic: %s, Correlation-id: %s", responsePublishTopic, msgEnvelope.CorrelationID)
			}
		}
	}()

	return nil
}
```

##### getCommand

```go
func getCommand(ctx context.Context, msgEnvelope types.MessageEnvelope, responseTopic string, deviceName string, commandName string, dic *di.Container) {
	var responseEnvelope types.MessageEnvelope
	
	lc := bootstrapContainer.LoggingClientFrom(dic.Get)
    // 获取messageBus
	messageBus := bootstrapContainer.MessagingClientFrom(dic.Get)
    // 获取请求
	rawQuery, reserved := filterQueryParams(msgEnvelope.QueryParams)

	// TODO: fix properly in EdgeX 3.0
	ctx = context.WithValue(ctx, common.CorrelationHeader, msgEnvelope.CorrelationID) // nolint: staticcheck
	event, edgexErr := application.GetCommand(ctx, deviceName, commandName, rawQuery, reserved[common.RegexCommand], dic)
    // 使用application.GetCommand发起请求，后面会分析application这个包
	if edgexErr != nil {
		lc.Errorf("Failed to process get device command %s for device %s: %s", commandName, deviceName, edgexErr.Error())
        // 如果有错误，构造一个带错误的responseEnvelope
		responseEnvelope = types.NewMessageEnvelopeWithError(msgEnvelope.RequestID, edgexErr.Error())
        // 对指定的topic，publish这个Envelope
		err := messageBus.Publish(responseEnvelope, responseTopic)
		if err != nil {
			lc.Errorf("Failed to publish command error response: %s", err.Error())
		}
		return
	}

	var err error
	var encoding string
	var eventResponseBytes []byte
	if reserved[common.ReturnEvent] {
        // 如果返回的结果是Event
        // 构造一个eventResponse
		eventResponse := responses.NewEventResponse(msgEnvelope.RequestID, "", http.StatusOK, *event)
		eventResponseBytes, encoding, err = eventResponse.Encode()
		if err != nil {
			lc.Errorf("Failed to encode event response: %s", err.Error())
			responseEnvelope = types.NewMessageEnvelopeWithError(msgEnvelope.RequestID, err.Error())
            // 向responsetopic 发起这个envelope
			err = messageBus.Publish(responseEnvelope, responseTopic)
			if err != nil {
				lc.Errorf("Failed to publish command error response: %s", err.Error())
			}
			return
		}
	} else {
        // 否则就不是Event结果，encoding结果就是TypeJSON
		eventResponseBytes = nil
		encoding = common.ContentTypeJSON
	}
	// 构造一个响应Envelope
	responseEnvelope, err = types.NewMessageEnvelopeForResponse(eventResponseBytes, msgEnvelope.RequestID, msgEnvelope.CorrelationID, encoding)
	if err != nil {
        // 如果有错，也会构造一个响应Envelope，不过会带上错误
		lc.Errorf("Failed to create response message envelope: %s", err.Error())
		responseEnvelope = types.NewMessageEnvelopeWithError(msgEnvelope.RequestID, err.Error())
        // 发送这个请求
		err = messageBus.Publish(responseEnvelope, responseTopic)
		if err != nil {
			lc.Errorf("Failed to publish command error response: %s", err.Error())
		}
		return
	}
	// 发起这个envelope的发布
	err = messageBus.Publish(responseEnvelope, responseTopic)
	if err != nil {
		lc.Errorf("Failed to publish command response: %s", err.Error())
		return
	}
	// 如果是PushEvent的话，
	if reserved[common.PushEvent] {
        // 开一个协程发送这个Event
		go sdkCommon.SendEvent(event, msgEnvelope.CorrelationID, dic)
	}

}
```

##### setCommand

```go
func setCommand(ctx context.Context, msgEnvelope types.MessageEnvelope, responseTopic string, deviceName string, commandName string, dic *di.Container) {
	var responseEnvelope types.MessageEnvelope

	lc := bootstrapContainer.LoggingClientFrom(dic.Get)
    // 这里来获取messageBus
	messageBus := bootstrapContainer.MessagingClientFrom(dic.Get)
	rawQuery, _ := filterQueryParams(msgEnvelope.QueryParams)

    // requestPayload 用来存放之前请求的结果
	requestPayload := make(map[string]any)
	err := json.Unmarshal(msgEnvelope.Payload, &requestPayload)
	if err != nil {
        // 如果出错了，要构造出错的响应Envelope
		lc.Errorf("Failed to decode set command request payload: %s", err.Error())
		responseEnvelope = types.NewMessageEnvelopeWithError(msgEnvelope.RequestID, err.Error())
		err = messageBus.Publish(responseEnvelope, responseTopic)
		if err != nil {
			lc.Errorf("Failed to publish command response: %s", err.Error())
		}
		return
	}

	// TODO: fix properly in EdgeX 3.0
	ctx = context.WithValue(ctx, common.CorrelationHeader, msgEnvelope.CorrelationID) // nolint: staticcheck
    // 使用application的SetCommand
	event, edgexErr := application.SetCommand(ctx, deviceName, commandName, rawQuery, requestPayload, dic)
	if edgexErr != nil {
        // 如果出错了，就构造出错的Envelope
		lc.Errorf("Failed to process set device command %s for device %s: %s", commandName, deviceName, edgexErr.Error())
		responseEnvelope = types.NewMessageEnvelopeWithError(msgEnvelope.RequestID, edgexErr.Error())
		err = messageBus.Publish(responseEnvelope, responseTopic)
		if err != nil {
			lc.Errorf("Failed to publish command response: %s", err.Error())
		}
		return
	}
	// 构造成功的Envelope
	responseEnvelope, err = types.NewMessageEnvelopeForResponse(nil, msgEnvelope.RequestID, msgEnvelope.CorrelationID, common.ContentTypeJSON)
	if err != nil {
        // 如果出错了，就构造出错的Envelope
		lc.Errorf("Failed to create response message envelope: %s", err.Error())
		responseEnvelope = types.NewMessageEnvelopeWithError(msgEnvelope.RequestID, err.Error())
		err = messageBus.Publish(responseEnvelope, responseTopic)
		if err != nil {
			lc.Errorf("Failed to publish command response: %s", err.Error())
		}
		return
	}
	// 发起这个topic的发布
	err = messageBus.Publish(responseEnvelope, responseTopic)
	if err != nil {
		lc.Errorf("Failed to publish command response: %s", err.Error())
		return
	}

	if event != nil {
        // 如果event不为空，就发送这个event
		go sdkCommon.SendEvent(event, msgEnvelope.CorrelationID, dic)
	}
}
```

#### MetadataSystemEventsCallback

```go
func MetadataSystemEventsCallback(ctx context.Context, dic *di.Container) errors.EdgeX {
	lc := bootstrapContainer.LoggingClientFrom(dic.Get)
    // 从容器中获取messagebusInfo和deviceservice
	messageBusInfo := container.ConfigurationFrom(dic.Get).MessageBus
	deviceService := container.DeviceServiceFrom(dic.Get)
    // 首先构建出需要订阅的topic
	metadataSystemEventTopic := common.BuildTopic(messageBusInfo.GetBaseTopicPrefix(),
		common.MetadataSystemEventSubscribeTopic, deviceService.Name, "#")

	lc.Infof("Subscribing to System Events on topic: %s", metadataSystemEventTopic)
	
    // 这些结构体是为了订阅内容
	messages := make(chan types.MessageEnvelope)
	messageErrors := make(chan error)
	topics := []types.TopicChannel{
		{
			Topic:    metadataSystemEventTopic,
			Messages: messages,
		},
	}

	messageBus := bootstrapContainer.MessagingClientFrom(dic.Get)
	err := messageBus.Subscribe(topics, messageErrors)
	if err != nil {
		return errors.NewCommonEdgeXWrapper(err)
	}

	go func() {
		for {
			select {
			case <-ctx.Done():
				lc.Infof("Exiting waiting for MessageBus '%s' topic messages", metadataSystemEventTopic)
				return
			case err = <-messageErrors:
				lc.Error(err.Error())
			case msgEnvelope := <-messages:
                // 如果来了消息以后
				lc.Debugf("System event received on message queue. Topic: %s, Correlation-id: %s", msgEnvelope.ReceivedTopic, msgEnvelope.CorrelationID)

				var systemEvent dtos.SystemEvent
                // 将消息转换为SystemEvent
				err := json.Unmarshal(msgEnvelope.Payload, &systemEvent)
				if err != nil {
					lc.Errorf("failed to JSON decoding system event: %s", err.Error())
					continue
				}

				serviceName := container.DeviceServiceFrom(dic.Get).Name
				if systemEvent.Owner != serviceName {
                    // 判断event的owner是不是这个service
					lc.Errorf("unmatched system event owner %s with service name %s", systemEvent.Owner, serviceName)
					continue
				}

				switch systemEvent.Type {
                // 如果是DeviceSystemEvent的话
				case common.DeviceSystemEventType:
					err = deviceSystemEventAction(systemEvent, dic)
					if err != nil {
						lc.Error(err.Error(), common.CorrelationHeader, msgEnvelope.CorrelationID)
					}
                // 如果是DeviceProfileSystemEvent的话
				case common.DeviceProfileSystemEventType:
					err = deviceProfileSystemEventAction(systemEvent, dic)
					if err != nil {
						lc.Error(err.Error(), common.CorrelationHeader, msgEnvelope.CorrelationID)
					}
                // 如果是ProvisionWatcherSystemEvent的话
				case common.ProvisionWatcherSystemEventType:
					err = provisionWatcherSystemEventAction(systemEvent, dic)
					if err != nil {
						lc.Error(err.Error(), common.CorrelationHeader, msgEnvelope.CorrelationID)
					}
                // 如果是DeviceServiceSystemEvent的话
				case common.DeviceServiceSystemEventType:
					err = deviceServiceSystemEventAction(systemEvent, dic)
					if err != nil {
						lc.Error(err.Error(), common.CorrelationHeader, msgEnvelope.CorrelationID)
					}
				default:
					lc.Errorf("unknown system event type %s", systemEvent.Type)
					continue
				}
			}
		}
	}()

	return nil
}
```

##### deviceSystemEventAction

```go
func deviceSystemEventAction(systemEvent dtos.SystemEvent, dic *di.Container) error {
	var device dtos.Device
	err := systemEvent.DecodeDetails(&device)
	if err != nil {
		return fmt.Errorf("failed to decode %s system event details: %s", systemEvent.Type, err.Error())
	}

	switch systemEvent.Action {
    // 增加的action
	case common.SystemEventActionAdd:
		err = application.AddDevice(requests.NewAddDeviceRequest(device), dic)
    // 更新的action
	case common.SystemEventActionUpdate:
		deviceModel := dtos.ToDeviceModel(device)
		updateDeviceDTO := dtos.FromDeviceModelToUpdateDTO(deviceModel)
		err = application.UpdateDevice(requests.NewUpdateDeviceRequest(updateDeviceDTO), dic)
    // 删除的action
	case common.SystemEventActionDelete:
		err = application.DeleteDevice(device.Name, dic)
	default:
		return fmt.Errorf("unknown %s system event action %s", systemEvent.Type, systemEvent.Action)
	}

	return err
}
```

##### deviceProfileSystemEventAction

```go
func deviceProfileSystemEventAction(systemEvent dtos.SystemEvent, dic *di.Container) error {
	var deviceProfile dtos.DeviceProfile
	err := systemEvent.DecodeDetails(&deviceProfile)
	if err != nil {
		return fmt.Errorf("failed to decode %s system event details: %s", systemEvent.Type, err.Error())
	}

	switch systemEvent.Action {
    // 更新的action
	case common.SystemEventActionUpdate:
		err = application.UpdateProfile(requests.NewDeviceProfileRequest(deviceProfile), dic)
	// there is no action needed for Device Profile Add and Delete in Device Service
    // 不需要有增加和删除的功能
	case common.SystemEventActionAdd, common.SystemEventActionDelete:
		break
	default:
		return fmt.Errorf("unknown %s system event action %s", systemEvent.Type, systemEvent.Action)
	}

	return err
}
```

##### provisionWatcherSystemEventAction

```go
func provisionWatcherSystemEventAction(systemEvent dtos.SystemEvent, dic *di.Container) error {
	var pw dtos.ProvisionWatcher
	err := systemEvent.DecodeDetails(&pw)
	if err != nil {
		return fmt.Errorf("failed to decode %s system event details: %s", systemEvent.Type, err.Error())
	}

	switch systemEvent.Action {
	case common.SystemEventActionAdd:
		err = application.AddProvisionWatcher(requests.NewAddProvisionWatcherRequest(pw), dic)
	case common.SystemEventActionUpdate:
		pwModel := dtos.ToProvisionWatcherModel(pw)
		pwDTO := dtos.FromProvisionWatcherModelToUpdateDTO(pwModel)
		err = application.UpdateProvisionWatcher(requests.NewUpdateProvisionWatcherRequest(pwDTO), dic)
	case common.SystemEventActionDelete:
		err = application.DeleteProvisionWatcher(pw.Name, dic)
	default:
		return fmt.Errorf("unknown %s system event action %s", systemEvent.Type, systemEvent.Action)
	}

	return err
}
```

##### deviceServiceSystemEventAction

```go
func deviceServiceSystemEventAction(systemEvent dtos.SystemEvent, dic *di.Container) error {
	var deviceService dtos.DeviceService
	err := systemEvent.DecodeDetails(&deviceService)
	if err != nil {
		return fmt.Errorf("failed to decode %s system event details: %s", systemEvent.Type, err.Error())
	}

	switch systemEvent.Action {
	case common.SystemEventActionUpdate:
		deviceServiceModel := dtos.ToDeviceServiceModel(deviceService)
		updateDeviceServiceDTO := dtos.FromDeviceServiceModelToUpdateDTO(deviceServiceModel)
		err = application.UpdateDeviceService(requests.NewUpdateDeviceServiceRequest(updateDeviceServiceDTO), dic)
	// there is no action needed for Device Service Add and Delete in Device Service
	case common.SystemEventActionAdd, common.SystemEventActionDelete:
		break
	default:
		return fmt.Errorf("unknown %s system event action %s", systemEvent.Type, systemEvent.Action)
	}

	return err
}
```

#### SubscribeDeviceValidation

```go
func SubscribeDeviceValidation(ctx context.Context, dic *di.Container) errors.EdgeX {
	lc := bootstrapContainer.LoggingClientFrom(dic.Get)
	messageBusInfo := container.ConfigurationFrom(dic.Get).MessageBus
	serviceName := container.DeviceServiceFrom(dic.Get).Name

    // 构造发起请求的topic
	requestTopic := common.BuildTopic(messageBusInfo.GetBaseTopicPrefix(), serviceName, common.ValidateDeviceSubscribeTopic)
	lc.Infof("Subscribing to device validation requests on topic: %s", requestTopic)
	
    // 这个是响应topic的前缀
	responseTopicPrefix := common.BuildTopic(messageBusInfo.GetBaseTopicPrefix(), common.ResponseTopic, serviceName)
	lc.Infof("Responses to device validation requests will be published on topic: %s/<requestId>", responseTopicPrefix)

	messages := make(chan types.MessageEnvelope)
	messageErrors := make(chan error)
	topics := []types.TopicChannel{
		{
			Topic:    requestTopic,
			Messages: messages,
		},
	}

	messageBus := bootstrapContainer.MessagingClientFrom(dic.Get)
	err := messageBus.Subscribe(topics, messageErrors)
	if err != nil {
		return errors.NewCommonEdgeXWrapper(err)
	}

	go func() {
		for {
			select {
			case <-ctx.Done():
				lc.Infof("Exiting waiting for MessageBus '%s' topic messages", requestTopic)
				return
			case err = <-messageErrors:
				lc.Error(err.Error())
			case msgEnvelope := <-messages:
				lc.Debugf("Device validation request received on message queue. Topic: %s, Correlation-id: %s", msgEnvelope.ReceivedTopic, msgEnvelope.CorrelationID)

				responseTopic := common.BuildTopic(responseTopicPrefix, msgEnvelope.RequestID)
				
                // 获取driver
				driver := container.ProtocolDriverFrom(dic.Get)

				var deviceRequest requests.AddDeviceRequest
                // 将内容填入driverRequest
				err = json.Unmarshal(msgEnvelope.Payload, &deviceRequest)
				if err != nil {
					lc.Errorf("Failed to JSON decoding AddDeviceRequest: %s", err.Error())
					res := types.NewMessageEnvelopeWithError(msgEnvelope.RequestID, err.Error())
					err = messageBus.Publish(res, responseTopic)
					if err != nil {
						lc.Errorf("Failed to publish device validation error response: %s", err.Error())
					}
					continue
				}
				
                // 用driver来验证device
				err = driver.ValidateDevice(dtos.ToDeviceModel(deviceRequest.Device))
				if err != nil {
					lc.Errorf("Device validation failed: %s", err.Error())
					res := types.NewMessageEnvelopeWithError(msgEnvelope.RequestID, err.Error())
					err = messageBus.Publish(res, responseTopic)
					if err != nil {
						lc.Errorf("Failed to publish device validation error response: %s", err.Error())
					}
					continue
				}

				res, err := types.NewMessageEnvelopeForResponse(nil, msgEnvelope.RequestID, msgEnvelope.CorrelationID, common.ContentTypeJSON)
				if err != nil {
					lc.Errorf("Failed to create device validation response envelope: %s", err.Error())
					continue
				}

				err = messageBus.Publish(res, responseTopic)
				if err != nil {
					lc.Errorf("Failed to publish device validation response: %s", err.Error())
					continue
				}

				lc.Debugf("Device validation response published on message queue. Topic: %s, Correlation-id: %s", responseTopic, msgEnvelope.CorrelationID)
			}
		}
	}()

	return nil
}
```

### NewServiceMetrics

> handlers.NewServiceMetrics(ds.ServiceName).BootstrapHandler：注册metrics reporter and manager，按照配置的interval，定期上报数据

```go
// BootstrapHandler fulfills the BootstrapHandler contract and performs initialization of service metrics.
// BootstrapHandler 履行 BootstrapHandler 协定并执行服务指标的初始化。
func (s *ServiceMetrics) BootstrapHandler(ctx context.Context, wg *sync.WaitGroup, _ startup.Timer, dic *di.Container) bool {
	lc := container.LoggingClientFrom(dic.Get)
  // 获取Service的设置
	serviceConfig := container.ConfigurationFrom(dic.Get)

  // 获取telemetry的配置，这个是用来干嘛的呢？
	telemetryConfig := serviceConfig.GetTelemetryInfo()

  // 如果Interval为空，就赋值为0s
	if telemetryConfig.Interval == "" {
		telemetryConfig.Interval = "0s"
	}

  // 获取时间间隔
	interval, err := time.ParseDuration(telemetryConfig.Interval)
	if err != nil {
		lc.Errorf("Telemetry interval is invalid time duration: %s", err.Error())
		return false
	}
	
  // 如果时间间隔为0，就赋值为最大值
	if interval == 0 {
		lc.Infof("0 specified for metrics reporting interval. Setting to max duration to effectively disable reporting.")
		interval = math.MaxInt64
	}

  // 构造基本topic
	baseTopic := serviceConfig.GetBootstrap().MessageBus.GetBaseTopicPrefix()
  // 构造一个新的reporter，
	reporter := metrics.NewMessageBusReporter(lc, baseTopic, s.serviceName, dic, telemetryConfig)
    // 根据interval 和 reporter 构造一个manager
	manager := metrics.NewManager(lc, interval, reporter)

	manager.Run(ctx, wg)
	
    // 更新manager？
	dic.Update(di.ServiceConstructorMap{
		container.MetricsManagerInterfaceName: func(get di.Get) interface{} {
			return manager
		},
	})

	return true
}
```

#### Manager

```go
// Run periodically (based on configured interval) reports the collected metrics using the configured MetricsReporter.
// 定期运行（基于配置的间隔）使用配置的指标报告程序报告收集的指标。
func (m *manager) Run(ctx context.Context, wg *sync.WaitGroup) {

	m.ticker = time.NewTicker(m.interval)

	wg.Add(1)
	defer wg.Done()

	go func() {
		for {
			select {
			case <-ctx.Done():
				m.lc.Info("Exited Metrics Manager Run...")
				return

			case <-m.ticker.C:
				m.tagsMutex.RLock()
                // tags是对metricTags的复制
				tags := copyTagMaps(m.metricTags)
				m.tagsMutex.RUnlock()
				
                // 时间到了，就定期上报收集到的指标
				if err := m.reporter.Report(m.registry, tags); err != nil {
					m.lc.Errorf(err.Error())
					continue
				}

				m.lc.Debug("Reported metrics...")
			}
		}
	}()

	m.lc.Infof("Metrics Manager started with a report interval of %s", m.interval.String())
}
```

#### Report

```go
// Report collects all the current metrics and reports them to the EdgeX MessageBus
// The approach here was adapted from https://github.com/vrischmann/go-metrics-influxdb
// 报告收集所有当前指标并将其报告给 EdgeX 消息总线
// 这里的方法改编自https://github.com/vrischmann/go-metrics-influxdb
func (r *messageBusReporter) Report(registry gometrics.Registry, metricTags map[string]map[string]string) error {
	var errs error
	publishedCount := 0

	// App Services create the messaging client after bootstrapping, so must get it from DIC when the first time
    // 获取messagebus的client
	if r.messageClient == nil {
		r.messageClient = container.MessagingClientFrom(r.dic.Get)
	}

	// If messaging client nil, then service hasn't set it up and can not report metrics this pass.
	// This may happen during bootstrapping if interval time is lower than time to bootstrap,
	// but will be resolved one messaging client has been added to the DIC.
    // 如果消息传递客户端为 nil，则服务尚未设置它，并且无法报告此传递的指标。
	// 如果间隔时间低于引导时间，则在引导过程中可能会发生这种情况，
	// 但将解决一个消息传递客户端已添加到 DIC。
	if r.messageClient == nil {
		return errors.New("messaging client not available. Unable to report metrics")
	}

	// Build the service tags each time we report since that can be changed in the Writable config
    // 每次报告时都构建服务标记，因为可以在可写配置中进行更改
	serviceTags := buildMetricTags(r.config.Tags)
	serviceTags = append(serviceTags, dtos.MetricTag{
		Name:  serviceNameTagKey,
		Value: r.serviceName,
	})

	registry.Each(func(itemName string, item interface{}) {
		var nextMetric dtos.Metric
		var err error

		// If itemName matches a configured Metric name, use the configured Metric name in case it is a partial match.
		// The metric item will have the extra name portion as a tag.
		// This is important for Metrics for App Service Pipelines, when the Metric name reported need to be the same
		// for all pipelines, but each will have to have unique name (with pipeline ID added) registered.
		// The Pipeline id will also be added as a tag.
        // 如果 itemName 与配置的指标名称匹配，使用配置的指标名称（如果它是部分匹配）。
		// 指标项将具有额外的名称部分作为标记。
		// 这对于应用服务管道的指标非常重要，因为报告的指标名称需要相同
		// 但每个管道都必须注册唯一的名称（添加了管道 ID）。
		// 管道 ID 也将添加为标记。
		name, isEnabled := r.config.GetEnabledMetricName(itemName)
		if !isEnabled {
			// This metric is not enable so do not report it.
			return
		}

		tags := append(serviceTags, buildMetricTags(metricTags[itemName])...)

		switch metric := item.(type) {
		case gometrics.Counter:
			snapshot := metric.Snapshot()
			fields := []dtos.MetricField{{Name: counterCountName, Value: snapshot.Count()}}
			nextMetric, err = dtos.NewMetric(name, fields, tags)

		case gometrics.Gauge:
			snapshot := metric.Snapshot()
			fields := []dtos.MetricField{{Name: gaugeValueName, Value: snapshot.Value()}}
			nextMetric, err = dtos.NewMetric(name, fields, tags)

		case gometrics.GaugeFloat64:
			snapshot := metric.Snapshot()
			fields := []dtos.MetricField{{Name: gaugeFloat64ValueName, Value: snapshot.Value()}}
			nextMetric, err = dtos.NewMetric(name, fields, tags)

		case gometrics.Timer:
			snapshot := metric.Snapshot()
			fields := []dtos.MetricField{
				{Name: timerCountName, Value: snapshot.Count()},
				{Name: timerMinName, Value: snapshot.Min()},
				{Name: timerMaxName, Value: snapshot.Max()},
				{Name: timerMeanName, Value: snapshot.Mean()},
				{Name: timerStddevName, Value: snapshot.StdDev()},
				{Name: timerVarianceName, Value: snapshot.Variance()},
			}
			nextMetric, err = dtos.NewMetric(name, fields, tags)

		case gometrics.Histogram:
			snapshot := metric.Snapshot()
			fields := []dtos.MetricField{
				{Name: histogramCountName, Value: snapshot.Count()},
				{Name: histogramMinName, Value: snapshot.Min()},
				{Name: histogramMaxName, Value: snapshot.Max()},
				{Name: histogramMeanName, Value: snapshot.Mean()},
				{Name: histogramStddevName, Value: snapshot.StdDev()},
				{Name: histogramVarianceName, Value: snapshot.Variance()},
			}
			nextMetric, err = dtos.NewMetric(name, fields, tags)

		default:
			errs = multierror.Append(errs, fmt.Errorf("metric type %T not supported", metric))
			return
		}

		if err != nil {
			err = fmt.Errorf("unable to create metric for '%s': %s", name, err.Error())
			errs = multierror.Append(errs, err)
			return
		}

		payload, err := json.Marshal(nextMetric)
		if err != nil {
			errs = multierror.Append(errs, fmt.Errorf("failed to marshal metric '%s' to JSON: %s", nextMetric.Name, err.Error()))
			return
		}

		message := types.MessageEnvelope{
			CorrelationID: uuid.NewString(),
			Payload:       payload,
			ContentType:   common.ContentTypeJSON,
		}

		topic := common.BuildTopic(r.baseMetricsTopic, name)
        // 进行信息的上报
		if err := r.messageClient.Publish(message, topic); err != nil {
			errs = multierror.Append(errs, fmt.Errorf("failed to publish metric '%s' to topic '%s': %s", name, topic, err.Error()))
			return
		} else {
			publishedCount++
		}
	})

	r.lc.Debugf("Publish %d metrics to the '%s' base topic", publishedCount, r.baseMetricsTopic)

	return errs
}
```

### clientBootstrapHandler

```go
// It creates instances of each of the EdgeX clients that are in the service's configuration and place them in the DIC.
// If the registry is enabled it will be used to get the URL for client otherwise it will use configuration for the url.
// This handler will fail if an unknown client is specified.
// 它创建服务配置中的每个 EdgeX 客户端的实例，并将它们放置在 DIC 中。
// 如果启用了注册表，它将用于获取客户端的 URL，否则它将使用 URL 的配置。
// 如果指定了未知客户端，则此处理程序将失败。
func (cb *ClientsBootstrap) BootstrapHandler(
	_ context.Context,
	_ *sync.WaitGroup,
	startupTimer startup.Timer,
	dic *di.Container) bool {

	lc := container.LoggingClientFrom(dic.Get)
    // 获取配置
	config := container.ConfigurationFrom(dic.Get)
	cb.registry = container.RegistryFrom(dic.Get)
	jwtSecretProvider := secret.NewJWTSecretProvider(container.SecretProviderFrom(dic.Get))
	
    // 遍历所有的client
	for serviceKey, serviceInfo := range config.GetBootstrap().Clients {
		var url string
		var err error

		if !serviceInfo.UseMessageBus {
            // 如果没有使用messageBus，那么就获取这个client对应的rest url
			url, err = cb.getClientUrl(serviceKey, serviceInfo.Url(), startupTimer, lc)
			if err != nil {
				lc.Error(err.Error())
				return false
			}
		}

		switch serviceKey {
        //根据serviceKey来区分使用的哪个client
		case common.CoreDataServiceKey:
			dic.Update(di.ServiceConstructorMap{
				container.EventClientName: func(get di.Get) interface{} {
					return clients.NewEventClient(url, jwtSecretProvider)
				},
			})
		case common.CoreMetaDataServiceKey:
			dic.Update(di.ServiceConstructorMap{
				container.DeviceClientName: func(get di.Get) interface{} {
					return clients.NewDeviceClient(url, jwtSecretProvider)
				},
				container.DeviceServiceClientName: func(get di.Get) interface{} {
					return clients.NewDeviceServiceClient(url, jwtSecretProvider)
				},
				container.DeviceProfileClientName: func(get di.Get) interface{} {
					return clients.NewDeviceProfileClient(url, jwtSecretProvider)
				},
				container.ProvisionWatcherClientName: func(get di.Get) interface{} {
					return clients.NewProvisionWatcherClient(url, jwtSecretProvider)
				},
			})

		case common.CoreCommandServiceKey:
            // command这边可能会使用messageBus
			var client interfaces.CommandClient

			if serviceInfo.UseMessageBus {
				// TODO: Move following outside loop when multiple messaging based clients exist
				messageClient := container.MessagingClientFrom(dic.Get)
				if messageClient == nil {
					lc.Errorf("Unable to create Command client using MessageBus: %s", "MessageBus Client was not created")
					return false
				}

				// TODO: Move following outside loop when multiple messaging based clients exist
				timeout, err := time.ParseDuration(config.GetBootstrap().Service.RequestTimeout)
				if err != nil {
					lc.Errorf("Unable to parse Service.RequestTimeout as a time duration: %v", err)
					return false
				}

				baseTopic := config.GetBootstrap().MessageBus.GetBaseTopicPrefix()
				client = clientsMessaging.NewCommandClient(messageClient, baseTopic, timeout)

				lc.Infof("Using messaging for '%s' clients", serviceKey)
			} else {
				client = clients.NewCommandClient(url, jwtSecretProvider)
			}

			dic.Update(di.ServiceConstructorMap{
				container.CommandClientName: func(get di.Get) interface{} {
					return client
				},
			})

		case common.SupportNotificationsServiceKey:
			dic.Update(di.ServiceConstructorMap{
				container.NotificationClientName: func(get di.Get) interface{} {
					return clients.NewNotificationClient(url, jwtSecretProvider)
				},
				container.SubscriptionClientName: func(get di.Get) interface{} {
					return clients.NewSubscriptionClient(url, jwtSecretProvider)
				},
			})

		case common.SupportSchedulerServiceKey:
			dic.Update(di.ServiceConstructorMap{
				container.IntervalClientName: func(get di.Get) interface{} {
					return clients.NewIntervalClient(url, jwtSecretProvider)
				},
				container.IntervalActionClientName: func(get di.Get) interface{} {
					return clients.NewIntervalActionClient(url, jwtSecretProvider)
				},
			})

		default:

		}
	}

	return true
}
```

### autoevent

#### executor

```go
type Executor struct {
    // 设备名称
	deviceName   string
    // 资源名称
	sourceName   string
    // 是否改变
	onChange     bool
	lastReadings map[string]interface{}
    // 持续时间
	duration     time.Duration
    // 是否停止
	stop         bool
    // 保证线程安全
	mutex        *sync.Mutex
}
```

```go
// Stop marks this Executor stopped
// Stop标志这个Executor停止
func (e *Executor) Stop() {
	e.stop = true
}
```

```go
// NewExecutor creates an Executor for an AutoEvent
// NewExecutor 为AutoEvent创建了一个Executor
func NewExecutor(deviceName string, ae models.AutoEvent) (*Executor, errors.EdgeX) {
	// check Frequency
    // 检查这个频率（其实就是检查Interval）
	duration, err := time.ParseDuration(ae.Interval)
	if err != nil {
		return nil, errors.NewCommonEdgeX(errors.KindServerError, fmt.Sprintf("failed to parse AutoEvent %s duration", ae.SourceName), err)
	}

	return &Executor{
		deviceName: deviceName,
		sourceName: ae.SourceName,
		onChange:   ae.OnChange,
		duration:   duration,
		stop:       false,
		mutex:      &sync.Mutex{}}, nil
}
```

```go
func (e *Executor) renewLastReadings(readings []dtos.BaseReading) {
    // 更新最后一次读取的内容
	e.lastReadings = make(map[string]interface{}, len(readings))
	for _, r := range readings {
		if r.ValueType == common.ValueTypeBinary 
            // 如果是这个ValueTypeBinary的Type，就加密么？
			e.lastReadings[r.ResourceName] = xxhash.Checksum64(r.BinaryValue)
		} else {
			e.lastReadings[r.ResourceName] = r.Value
		}
	}
}
// 总之这里是用来更新的
```

```go
func (e *Executor) compareReadings(readings []dtos.BaseReading) bool {
    // 这个应该是看自己的内容是不是最新的内容吧
	e.mutex.Lock()
	defer e.mutex.Unlock()

	if len(e.lastReadings) != len(readings) {
        // 如果长度不一样了
        // 如果不是，就更新，且返回false
		e.renewLastReadings(readings)
		return false
	}

	var result = true
    // 下面的话，就是根据情况，进行更新，如果更新了就返回false，否则返回true
	for _, reading := range readings {
		if lastReading, ok := e.lastReadings[reading.ResourceName]; ok {
			if reading.ValueType == common.ValueTypeBinary {
				checksum := xxhash.Checksum64(reading.BinaryValue)
				if lastReading != checksum {
					e.lastReadings[reading.ResourceName] = checksum
					result = false
				}
			} else {
				if lastReading != reading.Value {
					e.lastReadings[reading.ResourceName] = reading.Value
					result = false
				}
			}
		} else {
			e.renewLastReadings(readings)
			return false
		}
	}

	return result
}
```

```go
func readResource(e *Executor, dic *di.Container) (event *dtos.Event, err errors.EdgeX) {
    // 这里应该是发起请求，然后读取event内容吧
	vars := make(map[string]string, 2)
	vars[common.Name] = e.deviceName
	vars[common.Command] = e.sourceName

	res, err := application.GetCommand(context.Background(), e.deviceName, e.sourceName, "", true, dic)
	if err != nil {
		return event, err
	}
	return res, nil
}
```

```go
// Run triggers this Executor executes the handler for the event source periodically
// 运行触发器此执行程序定期执行事件源的处理程序
func (e *Executor) Run(ctx context.Context, wg *sync.WaitGroup, buffer chan bool, dic *di.Container) {
	wg.Add(1)
	defer wg.Done()

	lc := bootstrapContainer.LoggingClientFrom(dic.Get)
	for {
		select {
		case <-ctx.Done():
			return
            // 时间到了的话
		case <-time.After(e.duration):
			if e.stop {
                // 如果停止了，就直接返回
				return
			}
			lc.Debugf("AutoEvent - reading %s", e.sourceName)
            // 来读取资源
			evt, err := readResource(e, dic)
			if err != nil {
				lc.Errorf("AutoEvent - error occurs when reading resource %s: %v", e.sourceName, err)
				continue
			}

			if evt != nil {
				if e.onChange {
                    // 如果e被修改了
					if e.compareReadings(evt.Readings) {
                        // 判断是否相同，如果相同的话，就continue
						lc.Debugf("AutoEvent - readings are the same as previous one")
						continue
					}
				}
				// After the auto event executes a read command, it will create a goroutine to send out events.
                // 在自动事件执行读取命令后，它将创建一个goroutine来发送事件。
				// When the concurrent auto event amount becomes large, core-data might be hard to handle so many HTTP requests at the same time.
                // 当并发自动事件量变大时，核心数据可能难以同时处理如此多的 HTTP 请求。
				// The device service will get some network errors like EOF or Connection reset by peer.
                // 设备服务将收到一些网络错误，例如 EOF 或对等方重置连接。
				// By adding a buffer here, the user can use the Service.AsyncBufferSize configuration to control the goroutine for sending events.
                // 通过在此处添加缓冲区，用户可以使用 Service.AsyncBufferSize 配置来控制用于发送事件的 goroutine。
				go func() {
					buffer <- true
					correlationId := uuid.NewString()
                    // 开启一个协程，发送这个evt
                    // 这里可以再看一下这个SendEvent
					sdkCommon.SendEvent(evt, correlationId, dic)
					lc.Tracef("AutoEvent - Sent new Event/Reading for '%s' source with Correlation Id '%s'", evt.SourceName, correlationId)
					<-buffer
				}()
			} else {
				lc.Debugf("AutoEvent - no event generated when reading resource %s", e.sourceName)
			}
		}
	}
}
```

#### manager

```go
type manager struct {
    // 执行器的map
	executorMap     map[string][]*Executor
    // 上下文
	ctx             context.Context
	wg              *sync.WaitGroup
	mutex           sync.Mutex
	autoeventBuffer chan bool
	dic             *di.Container
}
```

```go
func (m *manager) StartAutoEvents() {
	m.mutex.Lock()
    // 锁住
	defer m.mutex.Unlock()

	for _, d := range cache.Devices().All() {
        // 遍历所有在cache中的device
		if _, ok := m.executorMap[d.Name]; !ok {
            // 如果对应的设备没有这个执行器的话,为他新建一组执行器
			executors := m.triggerExecutors(d.Name, d.AutoEvents, m.dic)
			m.executorMap[d.Name] = executors
		}
	}
}
```

```go
func (m *manager) triggerExecutors(deviceName string, autoEvents []models.AutoEvent, dic *di.Container) []*Executor {
	var executors []*Executor
	lc := bootstrapContainer.LoggingClientFrom(dic.Get)

	for _, autoEvent := range autoEvents {
        // 遍历所有的autoEvent，为每个autoEvent创建一个执行器
		executor, err := NewExecutor(deviceName, autoEvent)
		if err != nil {
			lc.Errorf("failed to create executor of AutoEvent %s for Device %s: %v", autoEvent.SourceName, deviceName, err)
			// skip this AutoEvent if it causes error during creation
			continue
		}
		executors = append(executors, executor)
        // 开一个协程，让这个控制器跑起来
		go executor.Run(m.ctx, m.wg, m.autoeventBuffer, dic)
	}
    // 最后返回这组执行器
	return executors
}
```

```go
func (m *manager) RestartForDevice(deviceName string) {
    // 重启设备
	lc := bootstrapContainer.LoggingClientFrom(m.dic.Get)
	
    // 将这个设备停止
	m.StopForDevice(deviceName)
    // 从cache中获取这个设备
	d, ok := cache.Devices().ForName(deviceName)
	if !ok {
		lc.Errorf("failed to find device %s in cache to start AutoEvent", deviceName)
	}

	m.mutex.Lock()
	defer m.mutex.Unlock()
    // 为这个设备新建执行器组
	executors := m.triggerExecutors(deviceName, d.AutoEvents, m.dic)
    // 更新这些执行器
	m.executorMap[deviceName] = executors
}
```

```go
func (m *manager) StopForDevice(deviceName string) {
	m.mutex.Lock()
	defer m.mutex.Unlock()

	executors, ok := m.executorMap[deviceName]
	if ok {
        // 将每个执行器停止
		for _, executor := range executors {
			executor.Stop()
		}
        // 删除控制器map中对应的设备名称
		delete(m.executorMap, deviceName)
	}
}
```

### NewBootstrap(router).BootstrapHandler

```go
func (b *Bootstrap) BootstrapHandler(ctx context.Context, wg *sync.WaitGroup, _ startup.Timer, dic *di.Container) (success bool) {
	s := b.deviceService
	s.wg = wg
	s.ctx = ctx
	s.lc = bootstrapContainer.LoggingClientFrom(dic.Get)
	s.autoEventManager = container.AutoEventManagerFrom(dic.Get)
	s.controller = http.NewRestController(b.router, dic, s.serviceKey)
    // 初始化RestRoutes
	s.controller.InitRestRoutes()
	
    // 初始化Cache
	edgexErr := cache.InitCache(s.serviceKey, dic)
	if edgexErr != nil {
		s.lc.Errorf("Failed to init cache: %s", edgexErr.Error())
		return false
	}
	
    // 是否允许异步读取
	if s.AsyncReadingsEnabled() {
        // 如果允许异步读取
		s.asyncCh = make(chan *models.AsyncValues, s.config.Device.AsyncBufferSize)
		wg.Add(1)
		go func() {
			defer wg.Done()
			s.processAsyncResults(ctx, dic)
		}()
	}
	
    // 是否允许设备发现
    // 这个应该是用于动态设备发现的
	if s.DeviceDiscoveryEnabled() {
		s.deviceCh = make(chan []models.DiscoveredDevice, 1)
		wg.Add(1)
		go func() {
			defer wg.Done()
            // 动态添加设备
			s.processAsyncFilterAndAdd(ctx)
		}()
	}

    // 初始化Initialize
	err := s.driver.Initialize(s)
	if err != nil {
		s.lc.Errorf("ProtocolDriver init failed: %s", err.Error())
		return false
	}

    // 将自己作为DeviceService注册到metadata中
	edgexErr = s.selfRegister()
	if edgexErr != nil {
		s.lc.Errorf("Failed to register %s on Metadata: %s", s.serviceKey, edgexErr.Error())
		return false
	}
	
    // 加载本地的profile
	edgexErr = provision.LoadProfiles(s.config.Device.ProfilesDir, dic)
	if edgexErr != nil {
		s.lc.Errorf("Failed to load device profiles: %s", edgexErr.Error())
		return false
	}

    // 加载本地的device
	edgexErr = provision.LoadDevices(s.config.Device.DevicesDir, dic)
	if edgexErr != nil {
		s.lc.Errorf("Failed to load devices: %s", edgexErr.Error())
		return false
	}
	
    // 加载ProvisionWatcher
	edgexErr = provision.LoadProvisionWatchers(s.config.Device.ProvisionWatchersDir, dic)
	if edgexErr != nil {
		s.lc.Errorf("Failed to load provision watchers: %s", edgexErr.Error())
		return false
	}
	
    // 开启在autoEvent中设置的执行器
	s.autoEventManager.StartAutoEvents()

	// Very important that this bootstrap handler is called after the NewServiceMetrics handler so
	// MetricsManager dependency has been created.
    // 在 NewServiceMetrics 处理程序之后调用此引导处理程序非常重要，以便创建 MetricsManager 依赖项。
	common.InitializeSentMetrics(s.lc, dic)
	return true
}
```

#### InitRestRoutes

```go
func (c *RestController) InitRestRoutes() {
    // 这里在初始化路由
	c.lc.Info("Registering v2 routes...")
	// router.UseEncodedPath() tells the router to match the encoded original path to the routes
	c.router.UseEncodedPath()

	lc := container.LoggingClientFrom(c.dic.Get)
	secretProvider := container.SecretProviderFrom(c.dic.Get)
	authenticationHook := handlers.AutoConfigAuthenticationFunc(secretProvider, lc)

	// common
	c.addReservedRoute(common.ApiPingRoute, c.Ping).Methods(http.MethodGet)
	c.addReservedRoute(common.ApiVersionRoute, authenticationHook(c.Version)).Methods(http.MethodGet)
	c.addReservedRoute(common.ApiConfigRoute, authenticationHook(c.Config)).Methods(http.MethodGet)
	// secret
	c.addReservedRoute(common.ApiSecretRoute, authenticationHook(c.Secret)).Methods(http.MethodPost)
	// discovery
	c.addReservedRoute(common.ApiDiscoveryRoute, authenticationHook(c.Discovery)).Methods(http.MethodPost)
	// device command
	c.addReservedRoute(common.ApiDeviceNameCommandNameRoute, authenticationHook(c.GetCommand)).Methods(http.MethodGet)
	c.addReservedRoute(common.ApiDeviceNameCommandNameRoute, authenticationHook(c.SetCommand)).Methods(http.MethodPut)

	c.router.Use(correlation.ManageHeader)
	c.router.Use(correlation.LoggingMiddleware(c.lc))
	c.router.Use(correlation.UrlDecodeMiddleware(c.lc))
}
```

#### cache.InitCache

```go
// InitCache Init basic state for cache
func InitCache(name string, dic *di.Container) errors.EdgeX {
	dc := bootstrapContainer.DeviceClientFrom(dic.Get)
	dpc := bootstrapContainer.DeviceProfileClientFrom(dic.Get)
	pwc := bootstrapContainer.ProvisionWatcherClientFrom(dic.Get)

	// init device cache
    // 初始化device cache
	deviceRes, err := dc.DevicesByServiceName(context.Background(), name, 0, -1)
	if err != nil {
		return err
	}
	devices := make([]models.Device, len(deviceRes.Devices))
	for i := range deviceRes.Devices {
		devices[i] = dtos.ToDeviceModel(deviceRes.Devices[i])
	}
	newDeviceCache(devices)

	// init profile cache
    // 初始化profile cache
	profiles := make([]models.DeviceProfile, len(devices))
	for i, d := range devices {
		res, err := dpc.DeviceProfileByName(context.Background(), d.ProfileName)
		if err != nil {
			return err
		}
		profiles[i] = dtos.ToDeviceProfileModel(res.Profile)
	}
	newProfileCache(profiles)

	// init provision watcher cache
    // 初始化provision watcher cache
	pwRes, err := pwc.ProvisionWatchersByServiceName(context.Background(), name, 0, -1)
	if err != nil {
		return err
	}
	pws := make([]models.ProvisionWatcher, len(pwRes.ProvisionWatchers))
	for i := range pwRes.ProvisionWatchers {
		pws[i] = dtos.ToProvisionWatcherModel(pwRes.ProvisionWatchers[i])
	}
	newProvisionWatcherCache(pws)

	return nil
}
```

#### processAsyncResults

```go
// processAsyncResults processes readings that are pushed from
// a DS implementation. Each is reading is optionally transformed
// before being pushed to Core Data.
// In this function, AsyncBufferSize is used to create a buffer for
// processing AsyncValues concurrently, so that events may arrive
// out-of-order in core-data / app service when AsyncBufferSize value
// is greater than or equal to two. Alternatively, we can process
// AsyncValues one by one in the same order by changing the AsyncBufferSize
// value to one.
// processAsyncResults处理从DS实现推送的读数。每个正在读取的数据在被推送到核心数据之前都有选择地进行转换。在此函数中，AsyncBufferSize用于创建一个缓冲区，用于同时处理AsyncValues，这样，当AsyncBufferSize值大于或等于2时，事件可能会在核心数据/应用程序服务中无序到达。或者，我们可以通过将AsyncBufferSize值更改为1，以相同的顺序逐个处理AsyncValues。
func (s *deviceService) processAsyncResults(ctx context.Context, dic *di.Container) {
	working := make(chan bool, s.config.Device.AsyncBufferSize)
	for {
		select {
		case <-ctx.Done():
			return
		case acv := <-s.asyncCh:
			go s.sendAsyncValues(acv, working, dic)
		}
	}
}
```

```go
// sendAsyncValues convert AsyncValues to event and send the event to CoreData
func (s *deviceService) sendAsyncValues(acv *sdkModels.AsyncValues, working chan bool, dic *di.Container) {
	working <- true
	defer func() {
		<-working
	}()

	if len(acv.CommandValues) == 0 {
		s.lc.Error("Skip sending AsyncValues because the CommandValues is empty.")
		return
	}
	if len(acv.CommandValues) > 1 && acv.SourceName == "" {
		s.lc.Error("Skip sending AsyncValues because the SourceName is empty.")
		return
	}
	// We can use the first reading's DeviceResourceName as the SourceName
	// when the CommandValues contains only one reading and the AsyncValues's SourceName is empty.
	if len(acv.CommandValues) == 1 && acv.SourceName == "" {
		acv.SourceName = acv.CommandValues[0].DeviceResourceName
	}

	configuration := container.ConfigurationFrom(dic.Get)
	event, err := transformer.CommandValuesToEventDTO(acv.CommandValues, acv.DeviceName, acv.SourceName, configuration.Device.DataTransform, dic)
	if err != nil {
		s.lc.Errorf("failed to transform CommandValues to Event: %v", err)
		return
	}

	common.SendEvent(event, "", dic)
}
```

#### InitializeSentMetrics

```go
func InitializeSentMetrics(lc logger.LoggingClient, dic *di.Container) {
    // 注册两个metrics 主要是eventsSent和readingsSent
	eventsSent = gometrics.NewCounter()
	readingsSent = gometrics.NewCounter()

	metricsManager := bootstrapContainer.MetricsManagerFrom(dic.Get)
	if metricsManager != nil {
		registerMetric(metricsManager, lc, eventsSentName, eventsSent)
		registerMetric(metricsManager, lc, readingsSentName, readingsSent)
	} else {
		lc.Warn("MetricsManager not available to register Event/Reading Sent metrics")
	}
}
```

```go
func registerMetric(metricsManager bootstrapInterfaces.MetricsManager, lc logger.LoggingClient, name string, metric interface{}) {
	err := metricsManager.Register(name, metric, nil)
	if err != nil {
		lc.Errorf("unable to register %s metric. Metric will not be reported: %v", name, err)
	} else {
		lc.Debugf("%s metric has been registered and will be reported (if enabled)", name)
	}
}
```

### autodiscovery.BootstrapHandler

```go
func BootstrapHandler(
	ctx context.Context,
	wg *sync.WaitGroup,
	_ startup.Timer,
	dic *di.Container) bool {
    // 自动设备发现相关的启动器，调用驱动实现的Discover()方法。
	driver := container.ProtocolDriverFrom(dic.Get)
	lc := bootstrapContainer.LoggingClientFrom(dic.Get)
	configuration := container.ConfigurationFrom(dic.Get)
	var runDiscovery bool = true

	if !configuration.Device.Discovery.Enabled {
		lc.Info("AutoDiscovery stopped: disabled by configuration")
		runDiscovery = false
	}
	duration, err := time.ParseDuration(configuration.Device.Discovery.Interval)
	if err != nil || duration <= 0 {
		lc.Info("AutoDiscovery stopped: interval error in configuration")
		runDiscovery = false
	}

	if runDiscovery {
		wg.Add(1)
		go func() {
			defer wg.Done()

			lc.Infof("Starting auto-discovery with duration %v", duration)
			DiscoveryWrapper(driver, lc)
			for {
				select {
				case <-ctx.Done():
					return
				case <-time.After(duration):
					DiscoveryWrapper(driver, lc)
				}
			}
		}()
	}

	return true
}
```

### handlers.NewStartMessage(serviceName, serviceVersion).BootstrapHandler

```go
// BootstrapHandler fulfills the BootstrapHandler contract.  It creates no go routines.  It logs a "standard" set of
// messages when the service first starts up successfully.
func (h StartMessage) BootstrapHandler(
	_ context.Context,
	_ *sync.WaitGroup,
	startupTimer startup.Timer,
	dic *di.Container) bool {
	
    // 打印整个驱动的启动提示信息，表示驱动起来了。
	lc := container.LoggingClientFrom(dic.Get)
	lc.Info("Service dependencies resolved...")
	lc.Info(fmt.Sprintf("Starting %s %s ", h.serviceKey, h.version))

	bootstrapConfig := container.ConfigurationFrom(dic.Get).GetBootstrap()
	if len(bootstrapConfig.Service.StartupMsg) > 0 {
		lc.Info(bootstrapConfig.Service.StartupMsg)
	}

	lc.Info("Service started in: " + startupTimer.SinceAsString())

	return true
}
```

**总结一下所有handler的作用功能**

1. httpServer.BootstrapHandler 主要负责注册TimeoutHandler、RequestLimitMiddleware、ProcessCORS handlers这些通用handler。
2. messageBusBootstrapHandler做的工作如下：

1. 1. 创建一个messageBus客户端
   2. 进行订阅，异步处理command请求（这个是由内部的application进行getcommand和setcommand处理的）
   3. 对metasystemevent的处理，分别处理deviceSystemEventAction、deviceProfileSystemEventAction、provisionWatcherSystemEventAction、deviceServiceSystemEventAction。这里对core-metadata进行了订阅。（虽然这里有实现了，但是好像还没有开始使用，因为在edgex官网上这条线是黄色的，是future时候使用的）
   4. 使用messagebus订阅设备的检验

1. handlers.NewServiceMetrics(ds.ServiceName).BootstrapHandler：注册metrics reporter and manager，按照配置的interval，reporter定期上报数据。但是这里没有注册metrics（在下面的handler中进行注册），只是将manager run起来了，reporter的上报是发给messageBus的。
2. handlers.NewClientsBootstrap().BootstrapHandler创建各种和其他服务的客户端。包括了：

1. 1. CoreData相关 -> EventClient （只有Rest）
   2. CoreMetaData相关 -> DeviceClient （只有Rest）、 DeviceServiceClient（只有Rest）、DeviceProfileClient（只有Rest）、ProvisionWatcherClient（只有Rest）
   3. CoreCommand相关 -> CommandClient （Rest 和 messageBus）
   4. SupportNotifications相关 -> NotificationClient （只有Rest）、SubscriptionClient（只有Rest）
   5. SupportScheduler相关 -> IntervalClient（只有Rest）、IntervalActionClient（只有Rest）

1. autoevent.BootstrapHandler 定义了autoEvent的manager，event的send是发给messageBus的（这里没有启动，留给后面的handler使用的）
2. NewBootstrap(router).BootstrapHandler 进行最后的初始化、加载、运行前面准备好的东西（autoevent的manager和注册需要上报的metric）

1. 1. 初始化路由：common、secret、discovery、validate、device command、callback相关的url
   2. 初始化cache：device、profile、provision watcher
   3. processAsyncResults异步读取，处理数据传到core-data之前，有的要做转换的地方。AsyncBufferSize > 1 乱序，AsyncBufferSize = 1，顺序。
   4. processAsyncFilterAndAdd处理添加动态设备
   5. 初始化、自己作为deviceservice、加载profile、加载device、加载provisionWatcher
   6. 启动之前设置好的autoevent的manager
   7. 为之前设置好的metricmanager注册两个metric（Event / Reading sent）

1. autodiscovery.BootstrapHandler：自动设备发现相关的启动器，调用驱动实现的Discover()方法。
2. handlers.NewStartMessage(serviceName, serviceVersion).BootstrapHandler：打印整个驱动的启动提示信息，表示驱动起来了。
