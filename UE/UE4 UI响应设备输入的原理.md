# UE4 UI响应设备输入的原理

## UE4引擎如何接受和处理输入

### 对输入设备的抽象
在UE4的源码当中，以Input开头的模块总共有两个，分别是`InputDevice`和`InputCore`，如果我们打开前者，在`InputDevice`这一模块中只定义了几个头文件，总结下来其实就两个文件是有有效信息的，他们分别是：
```cpp
// In module Runtime/InputDevice, file IInputDevice.h
class IInputDevice
{
public:
	virtual ~IInputDevice() {}

	virtual void Tick( float DeltaTime ) = 0;

	virtual void SendControllerEvents() = 0;

	virtual void SetMessageHandler( const TSharedRef< FGenericApplicationMessageHandler >& InMessageHandler ) = 0;

    virtual bool Exec( UWorld* InWorld, const TCHAR* Cmd, FOutputDevice& Ar ) = 0;

	virtual void SetChannelValue (int32 ControllerId, FForceFeedbackChannelType ChannelType, float Value) = 0;
	virtual void SetChannelValues (int32 ControllerId, const FForceFeedbackValues &values) = 0;
	virtual bool SupportsForceFeedback(int32 ControllerId) { return true; }

	virtual void SetLightColor(int32 ControllerId, FColor Color) { };
	virtual void ResetLightColor(int32 ControllerId) { };

	virtual void SetDeviceProperty(int32 ControllerId, const FInputDeviceProperty* Property) {}

	virtual class IHapticDevice* GetHapticDevice() { return nullptr; }

	virtual bool IsGamepadAttached() const { return false;}
};
```
以及
```cpp
// In module Runtime/InputDevice, file IHapticDevice.h
class IHapticDevice
{
public:
	virtual void SetHapticFeedbackValues(int32 ControllerId, int32 Hand, const FHapticFeedbackValues& Values) = 0;

	virtual void GetHapticFrequencyRange(float& MinFrequency, float& MaxFrequency) const = 0;

	virtual float GetHapticAmplitudeScale() const = 0;
};
```
能看到这两个I开头的接口类定义了一系列的纯虚函数，实际上这个接口定义的是针对所有能接入UE的输入设备的行为，如果我们在IDE中根据这两个接口查找其对应的Usage情况，能发现其在不同的平台中被使用了：
![](./images/InputDeviceResult.png)
如果我们把目光聚焦到Windows这个模块当中的使用，我们能发现其在两个文件中被使用到了，分别是：
* `XInputInterface.h`
* `WindowsApplication.cpp`

我们先来看第一个，这个文件中唯一干的一件事就是继承自`IInputDevice`这个基类，声明了一个`XInputInterface`，从其的注释我们能得知，这个InputInterface的子类，是为XBOX 360手柄所准备的一个输入设备类，除开在其中实现了`IInputDevice`当中的各种虚函数，还记录了XBOX手柄当中的：
```cpp
// In module Runtime/ApplicationCore/Windows, file XInputInterface.h
/**
 * Interface class for XInput devices (xbox 360 controller)                 
 */
class XInputInterface : public IInputDevice {
public:
	...
	virtual void SendControllerEvents() override;			// 发送按钮事件的函数
private:
	/** In the engine, all controllers map to xbox controllers for consistency */
	uint8 X360ToXboxControllerMapping[MAX_NUM_CONTROLLER_BUTTONS];			// 记录了按键映射

	FGamepadKeyNames::Type Buttons[MAX_NUM_CONTROLLER_BUTTONS];				// 记录了每一个按钮的名字

	TSharedRef<FGenericApplicationMessageHandler> MessageHandler;
};
```
而在这个类的其中，有一个比较关键的函数`SendControllerEvents`，我们来看一下里面究竟执行了什么东西：
```cpp
// In module Runtime/ApplicationCore/Windows, file XInputInterface.cpp
void XInputInterface::SendControllerEvents() {
	bool bWereConnected[MAX_NUM_XINPUT_CONTROLLERS];
	XINPUT_STATE XInputStates[MAX_NUM_XINPUT_CONTROLLERS];

	bIsGamepadAttached = false;
	for ( int32 ControllerIndex=0; ControllerIndex < MAX_NUM_XINPUT_CONTROLLERS; ++ControllerIndex ) {
		FControllerState& ControllerState = ControllerStates[ControllerIndex];

		bWereConnected[ControllerIndex] = ControllerState.bIsConnected;

		if( ControllerState.bIsConnected || bNeedsControllerStateUpdate ) {
			XINPUT_STATE& XInputState = XInputStates[ControllerIndex];
			FMemory::Memzero( &XInputState, sizeof(XINPUT_STATE) );

			ControllerState.bIsConnected = ( XInputGetState( ControllerIndex, &XInputState ) == ERROR_SUCCESS ) ? true : false;

			if (ControllerState.bIsConnected) {
				bIsGamepadAttached = true;
			}
		}
	}

	...
}
```
首先第一部分比较简单，就是检查当前的手柄是不是与电脑是连接了的状态，随后的第二部分则是挨个检查每一个案件的状态，并将其记录下来：
```cpp
// In module Runtime/ApplicationCore/Windows, file XInputInterface.cpp
void XInputInterface::SendControllerEvents() {
	...

	for ( int32 ControllerIndex = 0; ControllerIndex < MAX_NUM_XINPUT_CONTROLLERS; ++ControllerIndex ) {
		// Set input scope, there doesn't seem to be a reliable way to differentiate 360 vs Xbox one controllers so use generic name
		FInputDeviceScope InputScope(this, XInputInterfaceName, ControllerIndex, XInputControllerIdentifier);
		FControllerState& ControllerState = ControllerStates[ControllerIndex];
		const bool bWasConnected = bWereConnected[ControllerIndex];

		// If the controller is connected send events or if the controller was connected send a final event with default states so that 
		// the game doesn't think that controller buttons are still held down
		if( ControllerState.bIsConnected || bWasConnected ) {
			const XINPUT_STATE& XInputState = XInputStates[ControllerIndex];

			// If the controller is connected now but was not before, refresh the information
			if (!bWasConnected && ControllerState.bIsConnected) {
				FCoreDelegates::OnControllerConnectionChange.Broadcast(true, -1, ControllerState.ControllerId);
			} else if (bWasConnected && !ControllerState.bIsConnected) {
				FCoreDelegates::OnControllerConnectionChange.Broadcast(false, -1, ControllerState.ControllerId);
			}
			
			bool CurrentStates[MAX_NUM_CONTROLLER_BUTTONS] = {0};
		
			// 收集当前输入设备所有按键的状态
			CurrentStates[X360ToXboxControllerMapping[0]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_A);
			CurrentStates[X360ToXboxControllerMapping[1]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_B);
			CurrentStates[X360ToXboxControllerMapping[2]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_X);
			CurrentStates[X360ToXboxControllerMapping[3]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_Y);
			CurrentStates[X360ToXboxControllerMapping[4]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_LEFT_SHOULDER);
			CurrentStates[X360ToXboxControllerMapping[5]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_RIGHT_SHOULDER);
			CurrentStates[X360ToXboxControllerMapping[6]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_BACK);
			CurrentStates[X360ToXboxControllerMapping[7]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_START);
			CurrentStates[X360ToXboxControllerMapping[8]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_LEFT_THUMB);
			CurrentStates[X360ToXboxControllerMapping[9]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_RIGHT_THUMB);
			CurrentStates[X360ToXboxControllerMapping[10]] = !!(XInputState.Gamepad.bLeftTrigger > XINPUT_GAMEPAD_TRIGGER_THRESHOLD);
			CurrentStates[X360ToXboxControllerMapping[11]] = !!(XInputState.Gamepad.bRightTrigger > XINPUT_GAMEPAD_TRIGGER_THRESHOLD);
			CurrentStates[X360ToXboxControllerMapping[12]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_DPAD_UP);
			CurrentStates[X360ToXboxControllerMapping[13]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_DPAD_DOWN);
			CurrentStates[X360ToXboxControllerMapping[14]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_DPAD_LEFT);
			CurrentStates[X360ToXboxControllerMapping[15]] = !!(XInputState.Gamepad.wButtons & XINPUT_GAMEPAD_DPAD_RIGHT);
			CurrentStates[X360ToXboxControllerMapping[16]] = !!(XInputState.Gamepad.sThumbLY > XINPUT_GAMEPAD_LEFT_THUMB_DEADZONE);
			CurrentStates[X360ToXboxControllerMapping[17]] = !!(XInputState.Gamepad.sThumbLY < -XINPUT_GAMEPAD_LEFT_THUMB_DEADZONE);
			CurrentStates[X360ToXboxControllerMapping[18]] = !!(XInputState.Gamepad.sThumbLX < -XINPUT_GAMEPAD_LEFT_THUMB_DEADZONE);
			CurrentStates[X360ToXboxControllerMapping[19]] = !!(XInputState.Gamepad.sThumbLX > XINPUT_GAMEPAD_LEFT_THUMB_DEADZONE);
			CurrentStates[X360ToXboxControllerMapping[20]] = !!(XInputState.Gamepad.sThumbRY > XINPUT_GAMEPAD_RIGHT_THUMB_DEADZONE);
			CurrentStates[X360ToXboxControllerMapping[21]] = !!(XInputState.Gamepad.sThumbRY < -XINPUT_GAMEPAD_RIGHT_THUMB_DEADZONE);
			CurrentStates[X360ToXboxControllerMapping[22]] = !!(XInputState.Gamepad.sThumbRX < -XINPUT_GAMEPAD_RIGHT_THUMB_DEADZONE);
			CurrentStates[X360ToXboxControllerMapping[23]] = !!(XInputState.Gamepad.sThumbRX > XINPUT_GAMEPAD_RIGHT_THUMB_DEADZONE);

			...
		}
	}

	...
}
```
在收集完案件数据后，就要通过当前的按键状态来选择发出对应的事件了，在这一部分会将当前的按钮状态和之前的按钮状态进行对比，发出合适的事件，如`Pressed`或者`Released`事件。
```cpp
// In module Runtime/ApplicationCore/Windows, file XInputInterface.cpp
void XInputInterface::SendControllerEvents() {
	...
	for ( int32 ControllerIndex = 0; ControllerIndex < MAX_NUM_XINPUT_CONTROLLERS; ++ControllerIndex ) {
		...
		if( ControllerState.bIsConnected || bWasConnected ) {
			...
			// For each button check against the previous state and send the correct message if any
			for (int32 ButtonIndex = 0; ButtonIndex < MAX_NUM_CONTROLLER_BUTTONS; ++ButtonIndex) {
				// 比较状态，如需要则发送按键事件
				if (CurrentStates[ButtonIndex] != ControllerState.ButtonStates[ButtonIndex]) {
					if( CurrentStates[ButtonIndex] ) {
						MessageHandler->OnControllerButtonPressed( Buttons[ButtonIndex], ControllerState.ControllerId, false );
					} else {
						MessageHandler->OnControllerButtonReleased( Buttons[ButtonIndex], ControllerState.ControllerId, false );
					}

					if ( CurrentStates[ButtonIndex] != 0 ) {
						// this button was pressed - set the button's NextRepeatTime to the InitialButtonRepeatDelay
						ControllerState.NextRepeatTime[ButtonIndex] = CurrentTime + InitialButtonRepeatDelay;
					}
				} else if ( CurrentStates[ButtonIndex] != 0 && ControllerState.NextRepeatTime[ButtonIndex] <= CurrentTime ) {
					MessageHandler->OnControllerButtonPressed( Buttons[ButtonIndex], ControllerState.ControllerId, true );

					// set the button's NextRepeatTime to the ButtonRepeatDelay
					ControllerState.NextRepeatTime[ButtonIndex] = CurrentTime + ButtonRepeatDelay;
				}

				// Update the state for next time
				ControllerState.ButtonStates[ButtonIndex] = CurrentStates[ButtonIndex];
			}
			...	// 应用力反馈
		}
	}

	...
}
```
看完XBOX这个用来发送Event的函数了，现在我们有两点比较好奇：
1. 这个`SendControllerEvents`的函数究竟是在哪里会被调用？
2. 在这个类当中存在一个`MessageHandler`的成员在这个函数中也被使用到，并且是负责承担发送`OnControllerButtonPressed`等按键信息的一个成员，那这个`MessageHandler`究竟是什么？又是在哪里被设置的呢？

要解决第一个问题，我们就需要回到在Windows模块中使用到`IInputDevice`这个接口类的第二个文件——`WindowsApplication.cpp`上来，其只在`FWindowsApplication::PollGameDeviceState()`函数中被调用过，这个函数的定义非常的简短：
```cpp
// In module Runtime/ApplicationCore/Windows, file WindowsApplication.cpp
void FWindowsApplication::PollGameDeviceState( const float TimeDelta ) {
	if (bForceNoGamepads) { return; }

	// initialize any externally-implemented input devices (we delay load initialize the array so any plugins have had time to load)
	if (!bHasLoadedInputPlugins && GIsRunning ) {
		TArray<IInputDeviceModule*> PluginImplementations = IModularFeatures::Get().GetModularFeatureImplementations<IInputDeviceModule>( IInputDeviceModule::GetModularFeatureName() );
		for( auto InputPluginIt = PluginImplementations.CreateIterator(); InputPluginIt; ++InputPluginIt ) {
			TSharedPtr<IInputDevice> Device = (*InputPluginIt)->CreateInputDevice(MessageHandler);
			AddExternalInputDevice(Device);			
		}

		bHasLoadedInputPlugins = true;
	}

	if (FApp::UseVRFocus() && !FApp::HasVRFocus()) {
		return; // do not proceed if the app uses VR focus but doesn't have it
	}

	// Poll game device states and send new events
	XInput->SendControllerEvents();

	// Poll externally-implemented devices
	for( auto DeviceIt = ExternalInputDevices.CreateIterator(); DeviceIt; ++DeviceIt ) {
		(*DeviceIt)->Tick( TimeDelta );
		(*DeviceIt)->SendControllerEvents();
	}
}
```
我们忽略掉上半部分的加载部分，能看到在函数最下方的两个代码块，分别调用了`XInput`和`ExternalInputDevice`数组的`SendControllerEvents`函数，也就是我们在上面`XInputInterface`中了解到的用于检测当前按钮状态并且发送合适按钮事件的函数。这两个成员变量具体我们可以在`FWindowsApplication`的声明中找到：
```cpp
// In module Runtime/ApplicationCore/Windows, file WindowsApplication.h
/**
 * Windows-specific application implementation.
 */
class APPLICATIONCORE_API FWindowsApplication : public GenericApplication , public IForceFeedbackSystem {
	...
private:
	TSharedRef<class XInputInterface> XInput;
	/** List of input devices implemented in external modules. */
	TArray<TSharedPtr<class IInputDevice>> ExternalInputDevices;

	...
};
```
这里区分为了两种不同的输入设备类型，[一个是专门针对windows平台所使用的`XInput`，`XInput`是一个由微软提供的使得Windows应用程序能处理控制器交互的一个API，包括提供多个控制器的控制、设置震动效果和Deadzone等等的功能。](https://learn.microsoft.com/zh-cn/windows/win32/xinput/getting-started-with-xinput)
而另外的ExternalInputDevices则是用来处理除开XInput以外的外部输入设备事件。

现在，我们知道了这个`SendControllerEvents`在哪里被调用了，现在来看看其中的MessageHandler究竟是怎么被设置的，并且又是代表什么？我们回到`XInputInterface`的声明当中，能在其类声明的头部找到一个被覆写了的函数：
```cpp
void XInputInterface::SetMessageHandler( const TSharedRef< FGenericApplicationMessageHandler >& InMessageHandler ) {
	MessageHandler = InMessageHandler;
}
```
如果我们通过IDE的搜索功能进行使用搜索，我们能发现`MessageHandler`总共有两个地方被设置了，一个上面我们提到的`SetMessageHandler`函数，另一个就是在构造函数的时候进行赋值了。此时，我们只需要找到这个`XInputInterface`在哪里被构造出来、以及Setter函数在哪里被调用过，就能知道这个Handler究竟是什么东西了。

对应的，如果我们查找构造函数和Setter函数在哪里被调用了，同样只能在`FWindowsApplication`当中查找到，构造函数在`FWindowsApplication`的构造函数中通过调用静态函数`XInputInterface::Create( MessageHandler )`被调用，而Setter则是在`FWindowsApplication`中再次被包装了一层Setter函数——`FWindowsApplication::SetMessageHandler`：
```cpp
// In module Runtime/ApplicationCore/Windows, file WindowsApplication.cpp
// WindowsApplication构造函数
FWindowsApplication::FWindowsApplication( const HINSTANCE HInstance, const HICON IconHandle )
	: GenericApplication( MakeShareable( new FWindowsCursor() ) )
	...
	, XInput( XInputInterface::Create( MessageHandler ) )			// 设置MessageHandler
	...
	, ClipCursorRect() { ... }

...

void FWindowsApplication::SetMessageHandler( const TSharedRef< FGenericApplicationMessageHandler >& InMessageHandler ) {
	GenericApplication::SetMessageHandler(InMessageHandler);
	XInput->SetMessageHandler( InMessageHandler );				// 设置MessageHandler

	TArray<IInputDeviceModule*> PluginImplementations = IModularFeatures::Get().GetModularFeatureImplementations<IInputDeviceModule>( IInputDeviceModule::GetModularFeatureName() );
	for( auto DeviceIt = ExternalInputDevices.CreateIterator(); DeviceIt; ++DeviceIt ) {
		(*DeviceIt)->SetMessageHandler(InMessageHandler);		// 设置MessageHandler
	}
}
```
能看到，通过这个函数，所有和Windows当前这个应用相互连接的输入设备都被设置上了同一个MessageHandler。

对于MessageHandler我们先在此处暂停一下，总结一下上面的内容：在输入设备的抽象中，我们能找到其中存在着一个`SendControllerEvents`的函数，用于根据当前控制器的状态发送对应的输入事件；这个函数在`FWindowsApplication`中被调用；在这个函数当中，输入设备是用过`MessageHandler`来发送对应的输入事件的；`MessageHandler`的设置函数也是在`FWindowsApplication`中被调用；所有的输入设备的引用统一的管理在`FWindowsApplication`当中。
从上述的情况来看，似乎这个`FWindowsApplication`负责处理所有的输入事件，那这个`FWindowsApplication`究竟在整个引擎中代表了什么呢？

### `GenericApplication`
在`WindwosApplication.h`这个文件当中，我们能找到`FWindwosApplication`的继承关系，如下方代码所示：
```cpp
/**
 * Windows-specific application implementation.
 */
class APPLICATIONCORE_API FWindowsApplication : public GenericApplication, public IForceFeedbackSystem
{ ... };
```
从注释上我们可以得知，这个`FWindowsApplication`是一个针对Windows平台的、针对GenericApplication的一个实现。而当我们跳转到GenericApplication的声明中，我们能看到其声明中包含着大量的不执行任何操作的虚函数以定义接口。包括有创建窗口`MakeWindow`、获取窗口是否最小化`IsMinimized`等等的和窗口相关的操作；以及我们在上一个部分中非常关心的`PollGameDeviceState`和`SetMessageHandler`函数：
```cpp
/**
 * Generic platform application interface
 */
class GenericApplication 
{
public:

	DECLARE_MULTICAST_DELEGATE_OneParam( FOnConsoleCommandAdded, const FString& /*Command*/ );
	typedef FOnConsoleCommandAdded::FDelegate FOnConsoleCommandListener;

	GenericApplication( const TSharedPtr< ICursor >& InCursor )
		: Cursor( InCursor )
		, MessageHandler( MakeShareable( new FGenericApplicationMessageHandler() ) )
	{  }
	virtual ~GenericApplication() {}

	virtual void SetMessageHandler( const TSharedRef< FGenericApplicationMessageHandler >& InMessageHandler ) { MessageHandler = InMessageHandler; }

	TSharedRef< FGenericApplicationMessageHandler > GetMessageHandler() { return MessageHandler; }

#if WITH_ACCESSIBILITY
	virtual void SetAccessibleMessageHandler(const TSharedRef<FGenericAccessibleMessageHandler>& InAccessibleMessageHandler) { AccessibleMessageHandler = InAccessibleMessageHandler; }
	TSharedRef<FGenericAccessibleMessageHandler> GetAccessibleMessageHandler() const { return AccessibleMessageHandler; }
#endif

	virtual void PollGameDeviceState( const float TimeDelta ) { }
	virtual void PumpMessages( const float TimeDelta ) { }
	virtual void ProcessDeferredEvents( const float TimeDelta ) { }
	virtual void Tick ( const float TimeDelta ) { }
	virtual TSharedRef< FGenericWindow > MakeWindow() { return MakeShareable( new FGenericWindow() ); }
	virtual void InitializeWindow( const TSharedRef< FGenericWindow >& Window, const TSharedRef< FGenericWindowDefinition >& InDefinition, const TSharedPtr< FGenericWindow >& InParent, const bool bShowImmediately ) { }
	virtual void SetCapture( const TSharedPtr< FGenericWindow >& InWindow ) { }
	virtual void* GetCapture( void ) const { return NULL; }
	virtual FModifierKeysState GetModifierKeys() const  { return FModifierKeysState(); }

	/** @return true if the system cursor is currently directly over a slate window. */
	virtual bool IsCursorDirectlyOverSlateWindow() const { return true; }

	/** @return Native window under the mouse cursor. */
	virtual TSharedPtr< FGenericWindow > GetWindowUnderCursor() { return TSharedPtr< FGenericWindow >( nullptr ); }

	virtual bool IsMinimized() const { return false; }
	virtual void SetHighPrecisionMouseMode( const bool Enable, const TSharedPtr< FGenericWindow >& InWindow ) { };
	virtual bool IsUsingHighPrecisionMouseMode() const { return false; }
	virtual bool IsUsingTrackpad() const { return false; }
	virtual bool IsMouseAttached() const { return true; }
	virtual bool IsGamepadAttached() const { return false; }
	virtual void RegisterConsoleCommandListener(const FOnConsoleCommandListener& InListener) {}
	virtual void AddPendingConsoleCommand(const FString& InCommand) {}
	virtual FPlatformRect GetWorkArea( const FPlatformRect& CurrentWindow ) const {
		FPlatformRect OutRect;
		OutRect.Left = 0;
		OutRect.Top = 0;
		OutRect.Right = 0;
		OutRect.Bottom = 0;
		return OutRect;
	}

	virtual bool TryCalculatePopupWindowPosition( const FPlatformRect& InAnchor, const FVector2D& InSize, const FVector2D& ProposedPlacement, const EPopUpOrientation::Type Orientation, /*OUT*/ FVector2D* const CalculatedPopUpPosition ) const { return false; }

	DECLARE_EVENT_OneParam(GenericApplication, FOnDisplayMetricsChanged, const FDisplayMetrics&);
	
	/** Notifies subscribers when any of the display metrics change: e.g. resolution changes or monitor sare re-arranged. */
	FOnDisplayMetricsChanged& OnDisplayMetricsChanged(){ return OnDisplayMetricsChangedEvent; }

	virtual void GetInitialDisplayMetrics( FDisplayMetrics& OutDisplayMetrics ) const { FDisplayMetrics::RebuildDisplayMetrics(OutDisplayMetrics); }

	
	/** Delegate for virtual keyboard being shown/hidden in case UI wants to slide out of the way */
	DECLARE_EVENT_OneParam(FSlateApplication, FVirtualKeyboardShownEvent, FPlatformRect);
	FVirtualKeyboardShownEvent& OnVirtualKeyboardShown()  { return VirtualKeyboardShownEvent; }
	
	DECLARE_EVENT(FSlateApplication, FVirtualKeyboardHiddenEvent);
	FVirtualKeyboardHiddenEvent& OnVirtualKeyboardHidden()  { return VirtualKeyboardHiddenEvent; }

	
	/** Gets the horizontal alignment of the window title bar's title text. */
	virtual EWindowTitleAlignment::Type GetWindowTitleAlignment() const { return EWindowTitleAlignment::Left; }

	virtual EWindowTransparency GetWindowTransparencySupport() const { return EWindowTransparency::None; }

	virtual void DestroyApplication() { }

	virtual IInputInterface* GetInputInterface() { return nullptr; }	

	/** Function to return the current implementation of the Text Input Method System */
	virtual ITextInputMethodSystem *GetTextInputMethodSystem() { return NULL; }
	
	/** Send any analytics captured by the application */
	virtual void SendAnalytics(IAnalyticsProvider* Provider) { }
	virtual bool SupportsSystemHelp() const { return false; }
	virtual void ShowSystemHelp() {}
	virtual bool ApplicationLicenseValid(FPlatformUserId PlatformUser = PLATFORMUSERID_NONE) { return true; }
	virtual bool IsAllowedToRender() const { return true; }
	virtual void FinishedInputThisFrame() {}
public:

	const TSharedPtr< ICursor > Cursor;

protected:

	TSharedRef< class FGenericApplicationMessageHandler > MessageHandler;

	...
};
```
透过这些接口函数的声明，我们能大致了解到：`GenericApplication`是一个用于针对不同平台上创建窗口、管理窗口、管理输入设备的这么一个平台类，而`FWindowsApplication`则是一个针对Windows平台的一个用于管理窗口、管理输入设备的实现。

### `FGenericApplicationMessageHandler`包含了什么
这个Handler与其说是一个Handler，不如说是一个定义了一系列的按钮事件的接口，我们能在其声明中找到一系列定义为单纯的`return false;`的虚函数：
```cpp
/** Interface that defines how to handle interaction with a user via hardware input and output */
class FGenericApplicationMessageHandler
{
public:

	virtual ~FGenericApplicationMessageHandler() {}

	virtual bool ShouldProcessUserInputMessages( const TSharedPtr< FGenericWindow >& PlatformWindow ) const { return false; }

	virtual bool OnKeyChar( const TCHAR Character, const bool IsRepeat ) { return false; }

	virtual bool OnKeyDown( const int32 KeyCode, const uint32 CharacterCode, const bool IsRepeat ) { return false; }

	virtual bool OnKeyUp( const int32 KeyCode, const uint32 CharacterCode, const bool IsRepeat ) { return false; }

	virtual void OnInputLanguageChanged() { }

	virtual bool OnMouseDown( const TSharedPtr< FGenericWindow >& Window, const EMouseButtons::Type Button ) { return false; }

	virtual bool OnMouseDown( const TSharedPtr< FGenericWindow >& Window, const EMouseButtons::Type Button, const FVector2D CursorPos ) { return false; }

	virtual bool OnMouseUp( const EMouseButtons::Type Button ) { return false; }

	virtual bool OnMouseUp( const EMouseButtons::Type Button, const FVector2D CursorPos ) { return false; }

	virtual bool OnMouseDoubleClick( const TSharedPtr< FGenericWindow >& Window, const EMouseButtons::Type Button ) { return false; }

	virtual bool OnMouseDoubleClick( const TSharedPtr< FGenericWindow >& Window, const EMouseButtons::Type Button, const FVector2D CursorPos ) { return false; }

	...
};
```
能看到其中包含有一系列的按键事件的定义，包括`OnKeyDown`、`OnMouseDown`等等的用来检测键盘、鼠标和控制器的实践。

回到我们之前所看到的`XInputInterface`当中的`SendControllerEvents`函数，此时，我们已经能知道在这个XInput当中，是通过保存这一个输入事件传递的一个句柄，来将按钮事件给发送出去的了。

我们在此处看到的只是一个MessageHandler的定义，那么是在哪里对这些输入事件进行真正的处理的呢？

### 游戏本体的MessageHandler是怎么设置的（不包含编辑器部分）
透过IDE的搜索功能，我们能发现，在整个引擎当中，只有一个类真正的继承了`FGenericApplicationMessageHandler`——`FSlateAplication`。我们能看到在其中覆写了`FGenericApplicationMessageHandler`中的所有虚函数。

那这个MessageHandler究竟是什么时候被设置上的呢？回忆我们的上半部分，在`FWindowsApplication`中主要有两个途径对MessageHandler进行设置，分别是其构造函数和一个Setter函数。尝试针对`FWindwosApplication`的构造函数和Setter函数使用IDE中的搜索功能，我们能找到如下的结果：
```cpp
void FSlateApplication::SetPlatformApplication(const TSharedRef<class GenericApplication>& InPlatformApplication) {
	PlatformApplication->SetMessageHandler(MakeShareable(new FGenericApplicationMessageHandler()));
#if WITH_ACCESSIBILITY
	PlatformApplication->SetAccessibleMessageHandler(MakeShareable(new FGenericAccessibleMessageHandler()));
#endif

	PlatformApplication = InPlatformApplication;
	PlatformApplication->SetMessageHandler(CurrentApplication.ToSharedRef());
#if WITH_ACCESSIBILITY
	PlatformApplication->SetAccessibleMessageHandler(CurrentApplication->GetAccessibleMessageHandler());
#endif
}
```
以及，就在构造函数正上方的`CreateWindowsApplication`静态函数：
```cpp
FWindowsApplication* FWindowsApplication::CreateWindowsApplication( const HINSTANCE InstanceHandle, const HICON IconHandle ) {
	WindowsApplication = new FWindowsApplication( InstanceHandle, IconHandle );
	return WindowsApplication;
}
```
Emmm，构造函数这边我们似乎并没能获取到什么关于设置MessageHandler有用的信息，如果我们继续沿着Setter这条路径往上查找呢？
![alt text](./images/SetPlatformApplicationCaller.png)
更奇怪了，这个Caller甚至不在Runtime模块，而是和CL工具相关的调用，所以游戏本体的设置肯定不是在这里。

#### GenericApplication是保存在哪里的
既然通过Usage查询顺着设置函数和构造函数没有查出个所以然来，我们就先来解决一下另一个问题——这些用来控制窗口开启关闭的GenericApplication究竟是保存在哪里的？

在上面我们找到的`FSlateApplication::SetPlatformApplication`函数当中，我们能发现设置MessageHandler是调用的一个`PlatformApplication`的变量。跳转到其定义，我们能发现其是定义在`FSlateApplication`的基类之一`FSlateApplicationBase`的静态变量。
```cpp
// In module Runtime/SlateCore/Application/SlateApplicationBase.cpp
TSharedPtr<FSlateApplicationBase> FSlateApplicationBase::CurrentBaseApplication = nullptr;
TSharedPtr<GenericApplication> FSlateApplicationBase::PlatformApplication = nullptr;

...
```
既然这个`PlatformApplication`是保存在`SlateApplication`当中的，我们猜测一下，是不是MessageHandler会在`SlateApplication`创建的时候顺便被设置上呢？

#### 真正的给游戏本体设置MessageHandler的地方
对于一个创建对象的地方，一般来说都是在其构造函数当中。而在上一节中，我们已经发现了`SlateApplication`实际上是一个静态变量，通常而言，静态变量的创建是通过一个独立的静态Create函数来实现，顺着这个思路，我们能在`FSlateApplicaiton`当中找到两个不同的`FSlateApplication::Create`静态函数，他们的定义分别如下所示：
```cpp

void FSlateApplication::Create() {
	GSlateFastWidgetPath = GIsEditor ? false : true;

	Create(MakeShareable(FPlatformApplicationMisc::CreateApplication()));
}

TSharedRef<FSlateApplication> FSlateApplication::Create(const TSharedRef<class GenericApplication>& InPlatformApplication) {
	EKeys::Initialize();

	FCoreStyle::ResetToDefault();

	// Note: Important to establish the static PlatformApplication property first, as the FSlateApplication ctor relies on it
	PlatformApplication = InPlatformApplication;

	CurrentApplication = MakeShareable( new FSlateApplication() );	// 创建SlateApplication
	CurrentBaseApplication = CurrentApplication;

	PlatformApplication->SetMessageHandler( CurrentApplication.ToSharedRef() );	// 将当前的SlateApplication设置为GenericApplication的MessageHandler
#if WITH_ACCESSIBILITY
	PlatformApplication->SetAccessibleMessageHandler(CurrentApplication->GetAccessibleMessageHandler());
#endif

	// The grid needs to know the size and coordinate system of the desktop.
	// Some monitor setups have a primary monitor on the right and below the
	// left one, so the leftmost upper right monitor can be something like (-1280, -200)Synt
	{
		// Get an initial value for the VirtualDesktop geometry
		CurrentApplication->VirtualDesktopRect = []() {
			FDisplayMetrics DisplayMetrics;
			FSlateApplicationBase::Get().GetDisplayMetrics(DisplayMetrics);
			const FPlatformRect& VirtualDisplayRect = DisplayMetrics.VirtualDisplayRect;
			return FSlateRect(VirtualDisplayRect.Left, VirtualDisplayRect.Top, VirtualDisplayRect.Right, VirtualDisplayRect.Bottom);
		}();

		// Sign up for updates from the OS. Polling this every frame is too expensive on at least some OSs.
		PlatformApplication->OnDisplayMetricsChanged().AddSP(CurrentApplication.ToSharedRef(), &FSlateApplication::OnVirtualDesktopSizeChanged);
	}

	FAsyncTaskNotificationFactory::Get().RegisterFactory(TEXT("Slate"), []() -> FAsyncTaskNotificationFactory::FImplPointerType { return new FSlateAsyncTaskNotificationImpl(); });

	return CurrentApplication.ToSharedRef();
}
```
OK！终于找到我们想要的了！能看到在不带参数的Create函数中首先是调用了`MakeShareable(FPlatformApplicationMisc::CreateApplication())`创建了一个GenericApplication，随后将这个创建好的`GenericApplication`传递到下方的另一个Create函数，并且在其中用`MakeShareable(new FSlateApplication())`创建了一个新的SlateApplication，最后通过GenericApplication的`SetMessageHandler`将SlateApplication作为MessageHandler传递给了GenericApplication，随后发生的事情我们也知道了，在GenericApplication中会进一步的将MessageHandler传递到输入设备当中。

### 谁调用了SlateApplication的初始化和输入检测？
在上面的部分，我们已经知道了：通过调用`FSlateApplication::Create()`就可以把游戏运行起来所需要的SlateApplication和GenericApplication都给创建出来，那这个Create函数理应是在引擎初始化的时候就会被调用。我们可以通过搜索其引用一探究竟：
```cpp
int32 FEngineLoop::PreInitPreStartupScreen(const TCHAR* CmdLine) {
	...
	if (!IsRunningDedicatedServer() && (bHasEditorToken || bIsRegularClient)) {
		// Init platform application
		SCOPED_BOOT_TIMING("FSlateApplication::Create()");
		FSlateApplication::Create();		// 创建SlateApplication和GenericApplication
	}
	...
}
```
终于！我们找到了在引擎主循环当中的调用！在引擎主循环的`PreInitPreStartupScreen`函数当中、如果当前运行的不是专用服务器模式，并且如果是通常的客户端或者是拥有EditorToken的情况下，就会对Application进行创建。

既然他们的创建时在引擎主循环中创建的，那是不是意味着输入检测也是在主循环中呢？我们回到`FSlateApplication`的声明当中，还记得其中有一个函数就是负责进行输入检测的吗？就是`FSlateApplication::PollGameDeviceState()`，同样的，我们来看看这个函数在哪里被调用了。
![alt text](./images/PollGameDeviceState.png)
结果非常的简单，甚至不需要我们一个个排除，其代码段如下所示：
```cpp
void FEngineLoop::Tick() {
	...
	{
		...
		// process accumulated Slate input
		if (FSlateApplication::IsInitialized() && !bIdleMode) {
			CSV_SCOPED_TIMING_STAT_EXCLUSIVE(Input);
			SCOPE_TIME_GUARD(TEXT("SlateInput"));
			QUICK_SCOPE_CYCLE_COUNTER(STAT_FEngineLoop_Tick_SlateInput);
			LLM_SCOPE(ELLMTag::UI);

			FSlateApplication& SlateApp = FSlateApplication::Get();
			{
				QUICK_SCOPE_CYCLE_COUNTER(STAT_FEngineLoop_Tick_PollGameDeviceState);
				SlateApp.PollGameDeviceState();		// 检测输入
			}
			// Gives widgets a chance to process any accumulated input
			{
				QUICK_SCOPE_CYCLE_COUNTER(STAT_FEngineLoop_Tick_FinishedInputThisFrame);
				SlateApp.FinishedInputThisFrame();
			}
		}
		...
	}
	...
}
```
此时，我们已经可以下一个结论了，在引擎主循环初始化的时候，会将平台窗口管理器和SlateApplication创建出来；在之后引擎Tick函数中调用到检测输入的部分，意味着输入是每一帧中主动进行检测的；如果输入设备检测到状态变化了，则通过MessageHandler对应的输入事件发送出去，也就是调用到FSlateApplication中对应的事件。

## UE4 UI接收和处理输入并响应
我们在上面已经知道了：当按钮的状态改变后，比如上一帧某一个按钮的状态还是松开的，而当前帧则是按下的，此时就应该触发按键按下事件，而这个按下事件会被传递到`FSlateApplication`的对应事件当中。

以按键按下为例，当一个按钮按下后，最终会调用到`FSlateApplication::OnKeyDown`函数中：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::OnKeyDown( const int32 KeyCode, const uint32 CharacterCode, const bool IsRepeat ) {
	FKey const Key = FInputKeyManager::Get().GetKeyFromCodes( KeyCode, CharacterCode );
	FKeyEvent KeyEvent(Key, PlatformApplication->GetModifierKeys(), GetUserIndexForKeyboard(), IsRepeat, CharacterCode, KeyCode);

	return ProcessKeyDownEvent( KeyEvent );
}
```
能看到在这个函数中首先是通过KeyCode查找到对应的按键，并获取到其`FKey`结构体，随后构建出了一个`FKeyEvent`结构体，用于描述按键事件，最后将其传入`ProcessKeyDownEvent`函数当中，而`ProcessKeyDownEvent`函数的定义如下：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::ProcessKeyDownEvent( const FKeyEvent& InKeyEvent ) {
	SCOPE_CYCLE_COUNTER(STAT_ProcessKeyDown);

	TScopeCounter<int32> BeginInput(ProcessingInput);

	TSharedRef<FSlateUser> SlateUser = GetOrCreateUser(InKeyEvent);

	// Analog cursor gets first chance at the input
	if (InputPreProcessors.HandleKeyDownEvent(*this, InKeyEvent)) {
		return true;
	}

	FReply Reply = FReply::Unhandled();

	SetLastUserInteractionTime(this->GetCurrentTime());
	
	
	if (SlateUser->IsDragDropping() && InKeyEvent.GetKey() == EKeys::Escape)
	{
		// Pressing ESC while drag and dropping terminates the drag drop.
		SlateUser->CancelDragDrop();
		Reply = FReply::Handled();
	}
	else
	{
		LastUserInteractionTimeForThrottling = LastUserInteractionTime;

#if SLATE_HAS_WIDGET_REFLECTOR
		// If we are inspecting, pressing ESC exits inspection mode.
		if ( InKeyEvent.GetKey() == EKeys::Escape )
		{
			TSharedPtr<IWidgetReflector> WidgetReflector = WidgetReflectorPtr.Pin();
			const bool bIsWidgetReflectorPicking = WidgetReflector.IsValid() && WidgetReflector->IsInPickingMode();
			if ( bIsWidgetReflectorPicking )
			{
					WidgetReflector->OnWidgetPicked();
					Reply = FReply::Handled();

					return Reply.IsEventHandled();
			}
		}
#endif

#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
		// Ctrl+Shift+~ summons the Toolbox.
		if (InKeyEvent.GetKey() == EKeys::Tilde && InKeyEvent.IsControlDown() && InKeyEvent.IsShiftDown())
		{
			IToolboxModule* ToolboxModule = FModuleManager::LoadModulePtr<IToolboxModule>("Toolbox");
			if (ToolboxModule)
			{
				ToolboxModule->SummonToolbox();
			}
		}

#endif //!(UE_BUILD_SHIPPING || UE_BUILD_TEST)

		// Bubble the keyboard event
		TSharedRef<FWidgetPath> EventPathRef = SlateUser->GetFocusPath();
		const FWidgetPath& EventPath = EventPathRef.Get();

		// Switch worlds for widgets inOnPreviewMouseButtonDown the current path
		FScopedSwitchWorldHack SwitchWorld(EventPath);

		// Tunnel the keyboard event
		Reply = FEventRouter::RouteAlongFocusPath(this, FEventRouter::FTunnelPolicy(EventPath), InKeyEvent, [] (const FArrangedWidget& CurrentWidget, const FKeyEvent& Event) {
			if (CurrentWidget.Widget->IsEnabled()) {
				const FReply TempReply = CurrentWidget.Widget->OnPreviewKeyDown(CurrentWidget.Geometry, Event);
				return TempReply;
			}
			return FReply::Unhandled();
		}, ESlateDebuggingInputEvent::PreviewKeyDown);

		// Send out key down events.
		if ( !Reply.IsEventHandled() ) {
			Reply = FEventRouter::RouteAlongFocusPath(this, FEventRouter::FBubblePolicy(EventPath), InKeyEvent, [] (const FArrangedWidget& SomeWidgetGettingEvent, const FKeyEvent& Event) {
				if (SomeWidgetGettingEvent.Widget->IsEnabled()) {
					const FReply TempReply = SomeWidgetGettingEvent.Widget->OnKeyDown(SomeWidgetGettingEvent.Geometry, Event);
					return TempReply;
				}

				return FReply::Unhandled();
			}, ESlateDebuggingInputEvent::KeyDown);
		}

		// If the key event was not processed by any widget...
		if ( !Reply.IsEventHandled() && UnhandledKeyDownEventHandler.IsBound() )
		{
			Reply = UnhandledKeyDownEventHandler.Execute(InKeyEvent);
		}
	}

	return Reply.IsEventHandled();
}
```
要理解按键事件是怎么继续往下传递的，我们首先需要理解这个函数里面究竟做了什么工作。
首先，在这个函数中先通过了`GetOrCreateUser`函数，获取或者是创建了（在当前没有User的情况下）一个SlateUser，这个FSlateUser对象代表着一个逻辑输入用户，通常而言，一个单人游戏中就只会存在一个SlateUser对象，而实际上，这个SlateUser就是基于`OnKeyDown`函数中传入的`CharacterDown`参数所创建的。在创建完成后，通过使用FSlateUser的相关成员函数，我们能获取到当前逻辑用户的输入状态，比如在第一段代码中就检查了当前的输入是否是由于DragDrop操作触发的，如果是，则直接取消当前的拖拽操作，并把Reply标记为Handled。
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::ProcessKeyDownEvent( const FKeyEvent& InKeyEvent ) {
	...
	TSharedRef<FSlateUser> SlateUser = GetOrCreateUser(InKeyEvent);		// 获取当前的SlateUser
	if (SlateUser->IsDragDropping() && InKeyEvent.GetKey() == EKeys::Escape) {
		// Pressing ESC while drag and dropping terminates the drag drop.
		SlateUser->CancelDragDrop();
		Reply = FReply::Handled();
	} else {
		...
	}
	...
}
```
如果当前的状态不是正在拖拽，则会进入到下一个阶段，此时已经判断为当前的KeyDown是一个需要处理事件的输入了。在真正分析代码前，我们回想一下平时是怎么创建出一个复杂的UMG的：通常我们都是使用如CanvasPanel等容器，通过锚点等信息将其的子控件位置确定下来。也就是说屏幕上的某一个位置，可能有多个UMG堆叠在一起，比如下面的这种情况：
*[UButton -> UVerticalBox -> UCanvasPanel]*
那究竟UE是如何得知哪一个UMG需要处理相关的输入事件呢？

在UE中，使用一个叫做焦点路径的数据结构来管理，简而言之，就是UE能在每一帧中通过一定的规则，找到可能会被作为焦点的UMG控件，然后将他们通过从低到高的顺序串起来。
我们能在`ProcessKeyDownEvent`函数中找到获取焦点路径的部分：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::ProcessKeyDownEvent( const FKeyEvent& InKeyEvent ) {
	...
	TSharedRef<FSlateUser> SlateUser = GetOrCreateUser(InKeyEvent);		// 获取当前的SlateUser
	if (SlateUser->IsDragDropping() && InKeyEvent.GetKey() == EKeys::Escape) {
		...
	} else {
		...
		// Bubble the keyboard event
		TSharedRef<FWidgetPath> EventPathRef = SlateUser->GetFocusPath();
		const FWidgetPath& EventPath = EventPathRef.Get();
		...
	}
	...
}
```
### FocusPath（焦点路径）的构建过程
加入我们跳转到`FSlateUser::GetFocusPath()`函数的定义当中，我们能发现其是直接返回其中的一个`StrongFocusPath`的成员变量，进一步的查询这个成员变量在哪里被设置上了，我们就能在`FSlateApplication`中找到其两个相关的成员函数——`FSlateApplication::SetUserFocus()`，我们一段段的来看其中的代码，便能知道FocusPath的构建究竟是什么条件。
首先是第一段：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::SetUserFocus(FSlateUser& User, const FWidgetPath& InFocusPath, const EFocusCause InCause) {
	if (InFocusPath.IsValid()) {
		TSharedRef<SWindow> Window = InFocusPath.GetWindow();
		if (ActiveModalWindows.Num() != 0 && !(Window->IsDescendantOf(GetActiveModalWindow()) || ActiveModalWindows.Top() == Window)) {
			UE_LOG(LogSlate, Warning, TEXT("Ignoring SetUserFocus because it's not an active modal Window (user %i not set to %s."), User.GetUserIndex(), *InFocusPath.GetLastWidget()->ToString());
			return false;
		}
	}

	...
}
```
在这第一段代码中，首先检查了传入的`InFocusPath`是否有效，随后获取了这串`InFocusPath`所属的窗口，紧接着判断当前是否有激活的窗口、以及传入的`InFocusPath`所属的窗口是否是属于最顶层的窗口，如果都满足了，则表示这是一次合法的调用，否则将直接返回false。

接着是第二段代码：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::SetUserFocus(FSlateUser& User, const FWidgetPath& InFocusPath, const EFocusCause InCause) {
	...
	// Get the old Widget information
	const FWeakWidgetPath OldFocusedWidgetPath = User.GetWeakFocusPath();
	TSharedPtr<SWidget> OldFocusedWidget = OldFocusedWidgetPath.IsValid() ? OldFocusedWidgetPath.GetLastWidget().Pin() : TSharedPtr< SWidget >();
	
	// Get the new widget information by finding the first widget in the path that supports focus
	FWidgetPath NewFocusedWidgetPath;
	TSharedPtr<SWidget> NewFocusedWidget;

	if (InFocusPath.IsValid()) {
		for (int32 WidgetIndex = InFocusPath.Widgets.Num() - 1; WidgetIndex >= 0; --WidgetIndex) {
			const FArrangedWidget& WidgetToFocus = InFocusPath.Widgets[WidgetIndex];

			// Does this widget support keyboard focus?  If so, then we'll go ahead and set it!
			if (WidgetToFocus.Widget->SupportsKeyboardFocus()) {
				// Is we aren't changing focus then simply return
				if (WidgetToFocus.Widget == OldFocusedWidget) {
					//UE_LOG(LogSlate, Warning, TEXT("--Focus Has Not Changed--"));
					return false;
				}
				NewFocusedWidget = WidgetToFocus.Widget;
				NewFocusedWidgetPath = InFocusPath.GetPathDownTo(NewFocusedWidget.ToSharedRef());
				break;
			}
		}
	}
	...
}
```
这部分代码在从`InFocusPath`的最尾部，也就是在UMG中我们所设置的最叶子节点，往最根部遍历，然后挨个检查Widget是否支持键盘聚焦，如果是，并且这个最尾部Widget和上一次获取到的最尾部Widget不是同一个时，则直接将最新的聚焦Widget设置为找到的这个Widget，并且将`NewFocusedWidget`设置为`InFocusPath`从根到我们找到的最尾部且支持键盘聚焦的Widget这一段聚焦路径。
举个简单的例子来解释，当前传入的`InFocusPath`结构为：
[SWindow -> SPanel -> SButton -> STextBlock]
按照这个循环的规则，我们需要从最后往前进行遍历，所以先访问`STextBlock`对象，由于在`STextBlock`中并没有覆写`SWidget`中的`SupportsKeyboardFocus`虚函数，因此默认返回false，第一轮循环无事发生。
第二轮，需要访问到`SButton`对象了
```cpp
bool SButton::SupportsKeyboardFocus() const {
	// Buttons are focusable by default
	return bIsFocusable;
}
```
能看到按钮是否能支持键盘聚焦取决于按钮的设置，我们假设在这个例子中我们的按钮是Focusable的，此时我们在循环中的第一个if就会接收到一个true，并且我们假设上一次获取到的聚焦路径和这次的不同，此时：
1. `NewFocusedWidget`将被设置为`SButton`对象。
2. `NewFocusedWidgetPath`将被赋值为：[SWindow -> SPanel -> SButton]，把后面无法接受聚焦事件的`STextBlock`给剔除掉。

到第三段代码：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::SetUserFocus(FSlateUser& User, const FWidgetPath& InFocusPath, const EFocusCause InCause) {
	...
	User.IncrementFocusVersion();
	int32 CurrentFocusVersion = User.GetFocusVersion();
	FFocusEvent FocusEvent(InCause, User.GetUserIndex());
	FocusChangingDelegate.Broadcast(FocusEvent, OldFocusedWidgetPath, OldFocusedWidget, NewFocusedWidgetPath, NewFocusedWidget);

	// Notify widgets in the old focus path that focus is changing
	if (OldFocusedWidgetPath.IsValid()) {
		FScopedSwitchWorldHack SwitchWorld(OldFocusedWidgetPath.Window.Pin());

		for (int32 ChildIndex = 0; ChildIndex < OldFocusedWidgetPath.Widgets.Num(); ++ChildIndex) {
			TSharedPtr<SWidget> SomeWidget = OldFocusedWidgetPath.Widgets[ChildIndex].Pin();
			if (SomeWidget.IsValid()) {
				SomeWidget->OnFocusChanging(OldFocusedWidgetPath, NewFocusedWidgetPath, FocusEvent);

				// If focus setting is interrupted, stop what we're doing, as someone has already changed the focus path.
				if ( CurrentFocusVersion != User.GetFocusVersion()) {
					return false;
				}
			}
		}
	}

	// Notify widgets in the new focus path that focus is changing
	if (NewFocusedWidgetPath.IsValid()) {
		FScopedSwitchWorldHack SwitchWorld(NewFocusedWidgetPath.GetWindow());

		for (int32 ChildIndex = 0; ChildIndex < NewFocusedWidgetPath.Widgets.Num(); ++ChildIndex) {
			TSharedPtr<SWidget> SomeWidget = NewFocusedWidgetPath.Widgets[ChildIndex].Widget;
			if (SomeWidget.IsValid()) {
				SomeWidget->OnFocusChanging(OldFocusedWidgetPath, NewFocusedWidgetPath, FocusEvent);

				// If focus setting is interrupted, stop what we're doing, as someone has already changed the focus path.
				if ( CurrentFocusVersion != User.GetFocusVersion()) {
					return false;
				}
			}
		}
	}
	...
}
```
这段代码会将FocusPath出现变更的事件通过委托广播出去，包括整个FocusPath的变更以及前后两个FocusPath中的每一个Widget都要通知到，包括我们在C++和蓝图中所绑定的`OnAddedToFocusPath`和`OnRemovedFromFocusPath`事件。

到第四段代码：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::SetUserFocus(FSlateUser& User, const FWidgetPath& InFocusPath, const EFocusCause InCause) {
	...
	// Figure out if we should show focus for this focus entry
	bool ShowFocus = false;
	if (NewFocusedWidgetPath.IsValid()) {
		ShowFocus = InCause == EFocusCause::Navigation;
		for (int32 WidgetIndex = NewFocusedWidgetPath.Widgets.Num() - 1; WidgetIndex >= 0; --WidgetIndex) {
			TOptional<bool> QueryShowFocus = NewFocusedWidgetPath.Widgets[WidgetIndex].Widget->OnQueryShowFocus(InCause);
			if ( QueryShowFocus.IsSet()) {
				ShowFocus = QueryShowFocus.GetValue();
				break;
			}
		}
	}
	...
}
```
这段代码主要展示了如何从尾部遍历到跟组件寻找是否存在组件需要进行Focus Effect（焦点效果，比如当鼠标悬停在SButton上方会触发变底色等）的效果。以上面的例子来说：[SWindow -> SPanel -> SButton]在第一轮遍历中，如果SButton设置了相关的焦点效果，则循环直接Break出来，并且`ShowFocus`的值为true。

在完成焦点效果的查询后，就到了真正把焦点链设置到SlateUser当中去的时候了：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::SetUserFocus(FSlateUser& User, const FWidgetPath& InFocusPath, const EFocusCause InCause) {
	...
	// Store a weak widget path to the widget that's taking focus
	User.SetFocusPath(NewFocusedWidgetPath, InCause, ShowFocus);
	...
}
```
这里使用了传入的SlateUser的`SetFocusPath`函数，将新的焦点路径以及是否需要显示焦点效果等数据都传入到了SlateUser中。

继续往下，到第六段代码，也是这个函数的最后一个代码：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::SetUserFocus(FSlateUser& User, const FWidgetPath& InFocusPath, const EFocusCause InCause) {
	...
	// Let the old widget know that it lost keyboard focus
	if (OldFocusedWidget.IsValid()) {
		// Switch worlds for widgets in the old path
		FScopedSwitchWorldHack SwitchWorld(OldFocusedWidgetPath.Window.Pin());

		// Let previously-focused widget know that it's losing focus
		OldFocusedWidget->OnFocusLost(FocusEvent);
	}

	// Let the new widget know that it's received keyboard focus
	if (NewFocusedWidget.IsValid()) {
		TSharedPtr<SWindow> FocusedWindow = NewFocusedWidgetPath.GetWindow();

		// Switch worlds for widgets in the new path
		FScopedSwitchWorldHack SwitchWorld(FocusedWindow);

		// Set ActiveTopLevelWindow to the newly focused window
		ActiveTopLevelWindow = FocusedWindow;
		
		const FArrangedWidget& WidgetToFocus = NewFocusedWidgetPath.Widgets.Last();

		FReply Reply = NewFocusedWidget->OnFocusReceived(WidgetToFocus.Geometry, FocusEvent);
		if (Reply.IsEventHandled()) {
			ProcessReply(InFocusPath, Reply, nullptr, nullptr, User.GetUserIndex());
		}

		GetRelevantNavConfig(User.GetUserIndex())->OnNavigationChangedFocus(OldFocusedWidget, NewFocusedWidget, FocusEvent);
	}

	return true;
}
```
最后一段代码主要是调用了上一次焦点Widget的FocusLost事件，以及触发新焦点Widget的FocusReceived事件，最后函数结束返回true。
至此，我们已经知道了在`ProcessKeyDownEvent`中所获取到的FocusPath究竟是怎么样的了。

#### 传入`SetUserFocus`函数的`InFocusPath`参数又是怎么构建出来的？
我们查询`SetUserFocus`这个函数的调用者，能找到有好几个其他的同名重载函数，其中真正去获取一个链条的只有`bool FSlateApplication::SetUserFocus(uint32 UserIndex, const TSharedPtr<SWidget>& WidgetToFocus, EFocusCause ReasonFocusIsChanging)`这个重载，我们就来详细看看里面究竟都干了些什么。
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::SetUserFocus(uint32 UserIndex, const TSharedPtr<SWidget>& WidgetToFocus, EFocusCause ReasonFocusIsChanging /* = EFocusCause::SetDirectly*/) {
	TSharedPtr<FSlateUser> CurrentUser = GetUser(UserIndex);
	if (ensureMsgf(WidgetToFocus.IsValid(), TEXT("Attempting to focus an invalid widget. If your intent is to clear focus use ClearUserFocus()")) && CurrentUser) {
		FWidgetPath PathToWidget;
		const bool bFound = FSlateWindowHelper::FindPathToWidget(SlateWindows, WidgetToFocus.ToSharedRef(), /*OUT*/ PathToWidget);
		if (bFound) {
			return SetUserFocus(*CurrentUser, PathToWidget, ReasonFocusIsChanging);
		} else {
			const bool bFoundVirtual = FSlateWindowHelper::FindPathToWidget(SlateVirtualWindows, WidgetToFocus.ToSharedRef(), /*OUT*/ PathToWidget);
			if (bFoundVirtual) {
				return SetUserFocus(*CurrentUser, PathToWidget, ReasonFocusIsChanging);
			}
		}
	}

	return false;
}
```
能看到，透过一个`FSlateWindowHelper::FindPathToWidget`就能返回一个链条，其参数就是接收一个SlateWidget以及这个Widget所在的SlateWindow，那在这个函数里面又做了些什么工作呢？

##### SlateWidget的管理模式
在回答上面的问题之前，我们首先得知道UE是如何维护一个拥有着非常复杂嵌套关系的UI界面的。目光转到`SWidget`当中，我们能在其声明中找到如下的声明：
```cpp
class SLATECORE_API SWidget : public FSlateControlledConstruction, public TSharedFromThis<SWidget> {
public:
	FORCEINLINE bool IsParentValid() const { return ParentWidgetPtr.IsValid(); }
	FORCEINLINE TSharedPtr<SWidget> GetParentWidget() const { return ParentWidgetPtr.Pin(); }

	/**
	 * Every widget that has children must implement this method. This allows for iteration over the Widget's
	 * children regardless of how they are actually stored.
	 */
	virtual FChildren* GetChildren() = 0;
	virtual FChildren* GetAllChildren() { return GetChildren(); }

private:
	/** Pointer to this widgets parent widget.  If it is null this is a root widget or it is not in the widget tree */
	TWeakPtr<SWidget> ParentWidgetPtr;
};
```
也就是说，一个SWidget，会保存着其父Widget的指针，以及如果有孩子组件的话，也能通过实现`GetChildren`接口使得外部能获取到其所有孩子。一个父母、多个孩子，也就是说**SWidget的管理就是通过一颗树来维护的**。

##### 构建焦点路径点的预构建
在上面我们已经知道了，最终产生的焦点链条是通过另一条链条剪短而得到的，而这个原始链条则是通过传入的一个`WidgetToFocus`到函数`FSlateWindowHelper::FindPathToWidget`所获得的，现在来看一下这个函数：
```cpp
bool FSlateWindowHelper::FindPathToWidget( const TArray<TSharedRef<SWindow>>& WindowsToSearch, TSharedRef<const SWidget> InWidget, FWidgetPath& OutWidgetPath, EVisibility VisibilityFilter ) {
	SCOPE_CYCLE_COUNTER(STAT_FindPathToWidget);

	if (GSlateFastWidgetPath) {
		TSharedPtr<SWidget> CurWidget = ConstCastSharedRef<SWidget>(InWidget);
		OutWidgetPath.Widgets.SetFilter(VisibilityFilter);
		while (true) {
			EVisibility CurWidgetVisibility = CurWidget->GetVisibility();
			if (OutWidgetPath.Widgets.Accepts(CurWidgetVisibility)) {
				FArrangedWidget ArrangedWidget(CurWidget.ToSharedRef(), CurWidget->GetCachedGeometry());
				OutWidgetPath.Widgets.AddWidget(CurWidgetVisibility, ArrangedWidget);

				TSharedPtr<SWidget> CurWidgetParent = CurWidget->GetParentWidget();		// 获取父母Widget
				if (!CurWidgetParent.IsValid()) {
					if (CurWidget->Advanced_IsWindow()) {
						OutWidgetPath.TopLevelWindow = StaticCastSharedPtr<SWindow>(CurWidget);
						OutWidgetPath.Widgets.Reverse();
						return true;
					}

					OutWidgetPath.Widgets.Empty();
					return false;
				}

				if (!CurWidgetParent->ValidatePathToChild(CurWidget.Get())) {
					OutWidgetPath.Widgets.Empty();
					return false;
				}

				CurWidget = CurWidgetParent;		// 指针往上指一层
			} else {
				OutWidgetPath.Widgets.Empty();
				return false;
			}
		}
	} else {
		bool bFoundWidget = false;

		for (int32 WindowIndex = 0; !bFoundWidget && WindowIndex < WindowsToSearch.Num(); ++WindowIndex) {
			TSharedRef<SWindow> CurWindow = WindowsToSearch[WindowIndex];

			FArrangedChildren JustWindow(VisibilityFilter);
			{
				JustWindow.AddWidget(FArrangedWidget(CurWindow, CurWindow->GetWindowGeometryInScreen()));
			}

			FWidgetPath PathToWidget(CurWindow, JustWindow);

			if ((CurWindow == InWidget) || PathToWidget.ExtendPathTo(FWidgetMatcher(InWidget), VisibilityFilter)) {
				OutWidgetPath = PathToWidget;
				bFoundWidget = true;
			}

			if (!bFoundWidget) {
				bFoundWidget = FindPathToWidget(CurWindow->GetChildWindows(), InWidget, OutWidgetPath, VisibilityFilter);
			}
		}

		return bFoundWidget;
	}
}
```
虽然被一个if else statement分成了两个部分，但整体上他们都是干的一件事，只是区分是否全局支持快速搜索。在快速搜索部分，就是通过使用`SWidget::GetParentWidget()`函数获取当前Widget的父母Widget，然后将CurWidget设置为父母Widget，直到找到根Widget；而不支持快速搜索的情况下，则是通过递归的方式一层层的进行构建最终组合成一条路径。

### 按钮事件通过什么顺序传递到FocusPath中的Slate？
在`FSlateApplication::ProcessKeyDownEvent`函数获取到焦点路径后，我们能发现代码有两个非常相似的段落：
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::ProcessKeyDownEvent( const FKeyEvent& InKeyEvent ) {
	...
	TSharedRef<FSlateUser> SlateUser = GetOrCreateUser(InKeyEvent);		// 获取当前的SlateUser
	if (SlateUser->IsDragDropping() && InKeyEvent.GetKey() == EKeys::Escape) {
		...
	} else {
		...
		// Tunnel the keyboard event
		Reply = FEventRouter::RouteAlongFocusPath(this, FEventRouter::FTunnelPolicy(EventPath), InKeyEvent, [] (const FArrangedWidget& CurrentWidget, const FKeyEvent& Event) {
			if (CurrentWidget.Widget->IsEnabled()) {
				const FReply TempReply = CurrentWidget.Widget->OnPreviewKeyDown(CurrentWidget.Geometry, Event);
				return TempReply;
			}
			return FReply::Unhandled();
		}, ESlateDebuggingInputEvent::PreviewKeyDown);

		// Send out key down events.
		if ( !Reply.IsEventHandled() ) {
			Reply = FEventRouter::RouteAlongFocusPath(this, FEventRouter::FBubblePolicy(EventPath), InKeyEvent, [] (const FArrangedWidget& SomeWidgetGettingEvent, const FKeyEvent& Event) {
				if (SomeWidgetGettingEvent.Widget->IsEnabled()) {
					const FReply TempReply = SomeWidgetGettingEvent.Widget->OnKeyDown(SomeWidgetGettingEvent.Geometry, Event);
					return TempReply;
				}
				return FReply::Unhandled();
			}, ESlateDebuggingInputEvent::KeyDown);
		}
	}
	...
}
```
这段代码实质上就是在分别按照从前往后，以及从后往前的顺序遍历一遍FocusPath中所包含的Widget，并且调用`SWidget`当中对应的输入事件函数。
沿用之前的例子，假设我们现在的焦点路径为：
[SWindow -> SPanel -> SButton]
Tunneling方向遍历就是按照从根节点的SWindow开始往后遍历到SButton结束，而Bubbling则是从SButton出发，反向地往SWindow方向进行遍历。
这个设计类似于微软的WPF框架设计：先通过从根到叶的遍历，拦截诸如快捷键（例如Ctrl + S保存）等等的全局操作；再通过叶到根的遍历，给位于显示顶层的控件接收输入的机会。

在这个函数的最后，如果没有任何的Widget接收到当前的这个输入事件，则会将输入事件传递给位于`FSlateApplication`中定义的一个委托`UnhandledKeyDownEventHandler`，并且在最后，返回是否成功处理了本次输入事件。
```cpp
// In module Runtime/Slate/Framework/Application, file SlateApplication.cpp
bool FSlateApplication::ProcessKeyDownEvent( const FKeyEvent& InKeyEvent ) {
	TSharedRef<FSlateUser> SlateUser = GetOrCreateUser(InKeyEvent);		// 获取当前的SlateUser
	if (SlateUser->IsDragDropping() && InKeyEvent.GetKey() == EKeys::Escape) {
		...
	} else {
		...
		// If the key event was not processed by any widget...
		if ( !Reply.IsEventHandled() && UnhandledKeyDownEventHandler.IsBound() ) {
			Reply = UnhandledKeyDownEventHandler.Execute(InKeyEvent);
		} 
	}
	return Reply.IsEventHandled();
}
```

### Slate处理输入
从上面，我们已经知道了UE是通过先进行Tunneling在进行Bubbling、亦或者是其他的顺序来依次访问焦点路径中所包含的所有SlateWidget，然后调用Slate中所定义的一系列按键事件。

以我们鼠标在屏幕上点击一个Button为例子，此时在`FSlateApplication::ProcessMouseButtonDownEvent()`函数中就会调用到焦点链条中所包含的`SButton`的`SButton::OnMouseButtonDown()`函数，我们来看一下，这个函数中是如何处理输入的鼠标点击事件的：
```cpp
FReply SButton::OnMouseButtonDown( const FGeometry& MyGeometry, const FPointerEvent& MouseEvent ) {
	FReply Reply = FReply::Unhandled();
	if (IsEnabled() && (MouseEvent.GetEffectingButton() == EKeys::LeftMouseButton || MouseEvent.IsTouchEvent())) {
		Press();
		PressedScreenSpacePosition = MouseEvent.GetScreenSpacePosition();

		EButtonClickMethod::Type InputClickMethod = GetClickMethodFromInputType(MouseEvent);
		
		if(InputClickMethod == EButtonClickMethod::MouseDown) {
			//get the reply from the execute function
			Reply = ExecuteOnClick();

			//You should ALWAYS handle the OnClicked event.
			ensure(Reply.IsEventHandled() == true);
		} else if (InputClickMethod == EButtonClickMethod::PreciseClick) {
			// do not capture the pointer for precise taps or clicks
			// 
			Reply = FReply::Handled();
		} else {
			//we need to capture the mouse for MouseUp events
			Reply = FReply::Handled().CaptureMouse( AsShared() );
		}
	}

	Invalidate(EInvalidateWidget::Layout);

	//return the constructed reply
	return Reply;
}
```
这个函数中主要做了以下的几件事：
1. 首先创建了一个返回值`Reply`。
2. 随后判断：当前的按钮是否是Enabled，如果是，则判断传入的鼠标事件是否是用的左键触发、亦或者是这个MouseButtonDown是由触屏Touch事件模拟而来的。
3. 如果判断通过，则触发Press事件（后面会了解Press事件和Click事件所对应的函数），然后根据预设好的ClickMethod（用于指导触发规则，如Down或者是DownAndUp等）判断是否需要继续往下判断执行Click事件。
4. 最后，在处理完按键事件后，无效化当前的这个Button，这一步会触发按钮的重绘。然后返回Reply。

那`Press`和`ExecuteOnClick`函数当中又干了些什么呢？
```cpp
// In module Runtime/Slate/Widgets/Input, file SButton.h
/**
 * Slate's Buttons are clickable Widgets that can contain arbitrary widgets as its Content().
 */
class SLATE_API SButton : public SBorder {
	...
protected:
	...
	/** The delegate to execute when the button is clicked */
	FOnClicked OnClicked;

	/** The delegate to execute when the button is pressed */
	FSimpleDelegate OnPressed;

	...
};

// In module Runtime/Slate/Widgets/Input, file SButton.cpp
FReply SButton::ExecuteOnClick() {
	if (OnClicked.IsBound()) {
		FReply Reply = OnClicked.Execute();
		return Reply;
	} else {
		return FReply::Handled();
	}
}

void SButton::Press() {
	if ( !bIsPressed ) {
		bIsPressed = true;
		PlayPressedSound();
		OnPressed.ExecuteIfBound();
	}
}
```
在类的声明中，使用了`FOnClicked`和`FSimpleDelegate`，这两个都是不接受任何参数输入的简单的委托类型，其区别只是在`FOnClicked`存在一个`FReply`的返回值。
我们能看到，在用于处理Press和Click事件的函数中，分别调用了这两个委托所绑定的函数，用以执行对应的功能。那现在就有另一个问题了：这些诸如`OnPressed`的委托当中所绑定的函数究竟是在什么时候进行绑定的呢？我们对其进行搜索，能发现其赋值的唯一地方就是`SButton::Construct`函数当中，通过传入的`InArgs`进行赋值：
```cpp
// In module Runtime/Slate/Widgets/Input, file SButton.cpp
/**
 * Construct this widget
 *
 * @param	InArgs	The declaration data for this widget
 */
void SButton::Construct( const FArguments& InArgs ) {
	bIsPressed = false;

	...

	// 将传入的委托赋值给当前Slate当中的委托成员变量
	OnClicked = InArgs._OnClicked;
	OnPressed = InArgs._OnPressed;
	OnReleased = InArgs._OnReleased;
	OnHovered = InArgs._OnHovered;
	OnUnhovered = InArgs._OnUnhovered;

	ClickMethod = InArgs._ClickMethod;
	TouchMethod = InArgs._TouchMethod;
	PressMethod = InArgs._PressMethod;

	HoveredSound = InArgs._HoveredSoundOverride.Get(Style->HoveredSlateSound);
	PressedSound = InArgs._PressedSoundOverride.Get(Style->PressedSlateSound);
}
```
那么这个`Construct`函数又是在哪里，在什么时候被调用的呢？

#### Slate和UMG之间的关系
要回答上面的问题，首先就要说明`Slate`和`UUserWidget`之间存在什么关系。

我们都知道，平时在创建并制作`WidgetBlueprint`的时候，我们的Widget类实际上都是继承自`UUserWidget`下的某一个子类的，我们能从其声明中的注释中，以及真正的创建一个UMG类一窥：
```cpp
// In module Runtime/UMG/Blueprint, file UserWidget.h
/**
 * The user widget is extensible by users through the WidgetBlueprint.
 */
UCLASS(Abstract, editinlinenew, BlueprintType, Blueprintable, meta=( DontUseGenericSpawnObject="True", DisableNativeTick) )
class UMG_API UUserWidget : public UWidget, public INamedSlotInterface
{
	...
};
```
而对应的，我们在UMG中拖拽的诸如：图片、按钮之类的控件，则是直接继承自`UWidget`的一类控件，如按钮控件`UButton`继承自`UContentWidget`，`UContentWidget`继承自`UPanelWidget`，`UPanelWidget`则是继承自`UWidget`：
```cpp
// In module Runtime/UMG/Components, file Button.h
UCLASS()
class UMG_API UButton : public UContentWidget {
	...
};
```

嗯，既然UMG这边存在一个`UButton`来表示按钮，而在Slate中同样存在一个`SButton`来表示按钮并且处理针对的按钮的输入，那是不是说这二者之间存在这什么联系呢？
在`SButton`中，我们并没有发现和`UWidget`及其子类相关的任何内容，那反过来，我们是否能在`UWidget`当中找到和`SWidget`相关的内容呢？
还是以按钮组件为例，我们能在`UButton`当中找到如下内容：
```cpp
// In module Runtime/UMG/Components, file Button.h
UCLASS()
class UMG_API UButton : public UContentWidget {
	...
protected:
	//~ Begin UWidget Interface
	virtual TSharedRef<SWidget> RebuildWidget() override;		// 构建对应`SWidget`的接口函数
	...

protected:
	/** Cached pointer to the underlying slate button owned by this UWidget */
	TSharedPtr<SButton> MyButton;		// 指向SButton的指针
};

// In module Runtime/UMG/Components, file Button.cpp
TSharedRef<SWidget> UButton::RebuildWidget() {
	MyButton = SNew(SButton)
		.OnClicked(BIND_UOBJECT_DELEGATE(FOnClicked, SlateHandleClicked))
		.OnPressed(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandlePressed))
		.OnReleased(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandleReleased))
		.OnHovered_UObject( this, &ThisClass::SlateHandleHovered )
		.OnUnhovered_UObject( this, &ThisClass::SlateHandleUnhovered )
		.ButtonStyle(&WidgetStyle)
		.ClickMethod(ClickMethod)
		.TouchMethod(TouchMethod)
		.PressMethod(PressMethod)
		.IsFocusable(IsFocusable);

	if ( GetChildrenCount() > 0 ) {
		Cast<UButtonSlot>(GetContentSlot())->BuildSlot(MyButton.ToSharedRef());
	}
	
	return MyButton.ToSharedRef();
}
```
在这个类的声明定义当中，我们能发现一个指向`SButton`的指针，以及一个用来创建并赋值给`MuButton`的函数。

再看看其他我们经常用的控件——`UImage`
```cpp
// In module Runtime/UMG/Components, file Image.h
UCLASS()
class UMG_API UImage : public UWidget {
	...
protected:
	TSharedPtr<SImage> MyImage;			// 指向SImage的指针
	...
};
```
UMG层面的每一个控件，都对应着Slate层的相关控件，并且我们能看到Slate层并没有接入到UE的反射系统当中，也就是说：**UMG层实际上是针对Slate框架的，接入了UE的反射系统从而提供了蓝图相关功能的一层封装。**

#### Slate控件的创建
现在，我们已经知道了Slate和UMG之间的关系，回想上面的问题：我们想要找到究竟在哪里调用了`SWidget`的`Construct`函数。同时，我们也在UMG层面的实现中找到了`RebuildWidget`函数，其中就包含了看起来像是在构建一个新`SWidget`的部分：
```cpp
// In module Runtime/UMG/Components, file Button.cpp
TSharedRef<SWidget> UButton::RebuildWidget() {
	MyButton = SNew(SButton)
		.OnClicked(BIND_UOBJECT_DELEGATE(FOnClicked, SlateHandleClicked))
		.OnPressed(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandlePressed))
		.OnReleased(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandleReleased))
		.OnHovered_UObject( this, &ThisClass::SlateHandleHovered )
		.OnUnhovered_UObject( this, &ThisClass::SlateHandleUnhovered )
		.ButtonStyle(&WidgetStyle)
		.ClickMethod(ClickMethod)
		.TouchMethod(TouchMethod)
		.PressMethod(PressMethod)
		.IsFocusable(IsFocusable);

	if ( GetChildrenCount() > 0 ) {
		Cast<UButtonSlot>(GetContentSlot())->BuildSlot(MyButton.ToSharedRef());
	}
	
	return MyButton.ToSharedRef();
}
```
就来具体看一下这个`SNew`宏究竟干了些什么。具体往下展开，我们能看到如下的代码：
```cpp
// In module Runtime/SlateCore/Widgets/DeclaraiveSyntaxSupport.h
#define SNew( WidgetType, ... ) \
	MakeTDecl<WidgetType>( #WidgetType, __FILE__, __LINE__, RequiredArgs::MakeRequiredArgs(__VA_ARGS__) ) <<= TYPENAME_OUTSIDE_TEMPLATE WidgetType::FArguments()
```
这里有很多宏和模板的定义，把位于`UButton`中的`SNew`展开，我们能得到以下的代码：
```cpp
MakeTDecl<SButton>("SButton", "UButton.cpp", 53, RequiredArgs::MakeRequiredArgs()) 
	<<= 
	SButton::FArguments().OnClicked(BIND_UOBJECT_DELEGATE(FOnClicked, SlateHandleClicked))
				.OnClicked(BIND_UOBJECT_DELEGATE(FOnClicked, SlateHandleClicked))
				.OnPressed(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandlePressed))
				.OnReleased(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandleReleased))
				.OnHovered_UObject( this, &ThisClass::SlateHandleHovered )
				.OnUnhovered_UObject( this, &ThisClass::SlateHandleUnhovered )
				.ButtonStyle(&WidgetStyle)
				.ClickMethod(ClickMethod)
				.TouchMethod(TouchMethod)
				.PressMethod(PressMethod)
				.IsFocusable(IsFocusable);
```
看起来像是调用了一个叫做`MakeTDecl`的函数，然后针对这个函数的返回值使用了`<<=`操作符，且这个返回值返回的类型重载了`<<=`操作符传入了一个构造好的`WidgetType::FArguments()`。`MakeTDecl`的函数的函数以及其返回值的定义如下：
```cpp
// In module Runtime/SlateCore/Widgets/DeclaraiveSyntaxSupport.h
template<typename WidgetType, typename RequiredArgsPayloadType>
TDecl<WidgetType, RequiredArgsPayloadType> MakeTDecl( const ANSICHAR* InType, const ANSICHAR* InFile, int32 OnLine, RequiredArgsPayloadType&& InRequiredArgs ) {
	return TDecl<WidgetType, RequiredArgsPayloadType>(InType, InFile, OnLine, Forward<RequiredArgsPayloadType>(InRequiredArgs));
}

template<class WidgetType, typename RequiredArgsPayloadType>
struct TDecl {
	TDecl( const ANSICHAR* InType, const ANSICHAR* InFile, int32 OnLine, RequiredArgsPayloadType&& InRequiredArgs )
		: _Widget( TWidgetAllocator<WidgetType, TIsDerivedFrom<WidgetType, SUserWidget>::IsDerived >::PrivateAllocateWidget() )
		, _RequiredArgs(InRequiredArgs) {
		_Widget->SetDebugInfo( InType, InFile, OnLine, sizeof(WidgetType) );
	}

	...

	/**
	 * Complete widget construction from InArgs.
	 * @param InArgs  NamedArguments from which to construct the widget.
	 * @return A reference to the widget that we constructed.
	 */
	TSharedRef<WidgetType> operator<<=( const typename WidgetType::FArguments& InArgs ) const {
		...

		_RequiredArgs.CallConstruct(_Widget, InArgs);
		return _Widget;
	}

	const TSharedRef<WidgetType> _Widget;
	RequiredArgsPayloadType& _RequiredArgs;
};
```
能看到在`MakeTDecl`函数中，直接返回了一个构造好的`TDecl`对象，在`TDecl`的构造函数中，使用了传入进来的参数来设置其对应的调试信息，而在结构体定义的最下方，我们能找到我们想要的关于`<<=`的运算符重载，在其中把我们传入的已经构造好的`InArgs`调用了同样是我们传入其中的由`RequiredArgs::MakeRequiredArgs(__VA_ARGS__)`构造好的一个参数中的`CallConstruct`函数，随后，将构造好的`SWidget`返回回去，回到`UButton::RebuildWidget`当中，也就是将构造好的`SWidget`赋值给`MyButton`。
好了，大致的流程我们知道了，那这个传入的`RequiredArgs::MakeRequiredArgs(__VA_ARGS__)`究竟又是什么东西呢？我们跳转到这个函数对应的定义中，能发现其由很多个重载组合而成，他们分别接受不同数量的参数输入，从0个到5个：
```cpp
namespace RequiredArgs {
	...

	FORCEINLINE T0RequiredArgs MakeRequiredArgs() {
		return T0RequiredArgs();
	}

	template<typename Arg0Type>
	T1RequiredArgs<Arg0Type&&> MakeRequiredArgs(Arg0Type&& InArg0) {
		return T1RequiredArgs<Arg0Type&&>(Forward<Arg0Type>(InArg0));
	}

	template<typename Arg0Type, typename Arg1Type>
	T2RequiredArgs<Arg0Type&&, Arg1Type&&> MakeRequiredArgs(Arg0Type&& InArg0, Arg1Type&& InArg1) {
		return T2RequiredArgs<Arg0Type&&, Arg1Type&&>(Forward<Arg0Type>(InArg0), Forward<Arg1Type>(InArg1));
	}

	template<typename Arg0Type, typename Arg1Type, typename Arg2Type>
	T3RequiredArgs<Arg0Type&&, Arg1Type&&, Arg2Type&&> MakeRequiredArgs(Arg0Type&& InArg0, Arg1Type&& InArg1, Arg2Type&& InArg2) {
		return T3RequiredArgs<Arg0Type&&, Arg1Type&&, Arg2Type&&>(Forward<Arg0Type>(InArg0), Forward<Arg1Type>(InArg1), Forward<Arg2Type>(InArg2));
	}

	template<typename Arg0Type, typename Arg1Type, typename Arg2Type, typename Arg3Type>
	T4RequiredArgs<Arg0Type&&, Arg1Type&&, Arg2Type&&, Arg3Type&&> MakeRequiredArgs(Arg0Type&& InArg0, Arg1Type&& InArg1, Arg2Type&& InArg2, Arg3Type&& InArg3) {
		return T4RequiredArgs<Arg0Type&&, Arg1Type&&, Arg2Type&&, Arg3Type&&>(Forward<Arg0Type>(InArg0), Forward<Arg1Type>(InArg1), Forward<Arg2Type>(InArg2), Forward<Arg3Type>(InArg3));
	}

	template<typename Arg0Type, typename Arg1Type, typename Arg2Type, typename Arg3Type, typename Arg4Type>
	T5RequiredArgs<Arg0Type&&, Arg1Type&&, Arg2Type&&, Arg3Type&&, Arg4Type&&> MakeRequiredArgs(Arg0Type&& InArg0, Arg1Type&& InArg1, Arg2Type&& InArg2, Arg3Type&& InArg3, Arg4Type&& InArg4) {
		return T5RequiredArgs<Arg0Type&&, Arg1Type&&, Arg2Type&&, Arg3Type&&, Arg4Type&&>(Forward<Arg0Type>(InArg0), Forward<Arg1Type>(InArg1), Forward<Arg2Type>(InArg2), Forward<Arg3Type>(InArg3), Forward<Arg4Type>(InArg4));
	}
}
```
能看到，他们分别构造了针对不同数量入参的结构体然后返回，也就是说从`T0RequiredArgs`到`T5RequiredArgs`他们的定义都差不多，那我们就以其中的一个来作为参考，看看里面究竟都干了些什么：
```cpp
namespace RequiredArgs {
	...
	struct T0RequiredArgs {
		T0RequiredArgs() { }

		template<class WidgetType>
		void CallConstruct(const TSharedRef<WidgetType>& OnWidget, const typename WidgetType::FArguments& WithNamedArgs) const {
			// YOUR WIDGET MUST IMPLEMENT void Construct(const FArguments& InArgs)
			OnWidget->Construct(WithNamedArgs);
			OnWidget->CacheVolatility();
		}
	};

	...
}
```
能看到在对应的`CallConstruct`函数当中，接受了一个用来构造的Widget引用，以及在`<<=`运算符中传入的`InArgs`。到这里，我们已经破案了，上面我们苦苦寻找的`SWidget::Construct(const FArguments& InArgs)`的调用者就在这里。

距离真正理解`SNew`这个宏已经很接近了，在宏展开后，我们仍然有一个部分没有完全理解，就是`<<=`运算符后面的`WidgetType::FArguments()`，对其展开后就是针对特定类型的SWidget的一个FArgument，这意味着这个构造出来的结构体大概率就是每一个SWidget都会单独定义的一个内部结构体，我们到`SButton`中找一下：
```cpp
class SLATE_API SButton : public SBorder {
	SLATE_BEGIN_ARGS( SButton )
		: _Content()
		, _ButtonStyle( &FCoreStyle::Get().GetWidgetStyle< FButtonStyle >( "Button" ) )
		, _TextStyle( &FCoreStyle::Get().GetWidgetStyle< FTextBlockStyle >("NormalText") )
		, _HAlign( HAlign_Fill )
		, _VAlign( VAlign_Fill )
		, _ContentPadding(FMargin(4.0, 2.0))
		, _Text()
		, _ClickMethod( EButtonClickMethod::DownAndUp )
		, _TouchMethod( EButtonTouchMethod::DownAndUp )
		, _PressMethod( EButtonPressMethod::DownAndUp )
		, _DesiredSizeScale( FVector2D(1,1) )
		, _ContentScale( FVector2D(1,1) )
		, _ButtonColorAndOpacity(FLinearColor::White)
		, _ForegroundColor( FCoreStyle::Get().GetSlateColor( "InvertedForeground" ) )
		, _IsFocusable( true ) { }
		
		...
	
		/** Called when the button is clicked */
		SLATE_EVENT( FOnClicked, OnClicked )

		/** Called when the button is pressed */
		SLATE_EVENT( FSimpleDelegate, OnPressed )

		/** Called when the button is released */
		SLATE_EVENT( FSimpleDelegate, OnReleased )

		SLATE_EVENT( FSimpleDelegate, OnHovered )

		SLATE_EVENT( FSimpleDelegate, OnUnhovered )

		...

	SLATE_END_ARGS()

	...
};
```
为了方便定义，UE使用了一套专门的宏定义来简化各种函数的编写，在`SButton::FArgument`中，我们能看到一大堆和按钮相关的参数包括对齐、颜色和又名都、按钮当中的内容（按钮中间可以容纳一个Text或者是Image）等，当然也包括用于处理输入事件的几个委托。以OnPressed事件为例，我们将整个`SButton::FArgument`展开，会得到：
```cpp
class SLATE_API SButton : public SBorder {
public:
	struct FArguments : public TSlateBaseNamedArgs<SButton> {
		typedef FArguments WidgetArgsType;
		__declspec(noinline) FArguments()
			: _Content()
			, _ButtonStyle( &FCoreStyle::Get().GetWidgetStyle< FButtonStyle >( "Button" ) )
			, _TextStyle( &FCoreStyle::Get().GetWidgetStyle< FTextBlockStyle >("NormalText") )
			, _HAlign( HAlign_Fill )
			, _VAlign( VAlign_Fill )
			, _ContentPadding(FMargin(4.0, 2.0))
			, _Text()
			, _ClickMethod( EButtonClickMethod::DownAndUp )
			, _TouchMethod( EButtonTouchMethod::DownAndUp )
			, _PressMethod( EButtonPressMethod::DownAndUp )
			, _DesiredSizeScale( FVector2D(1,1) )
			, _ContentScale( FVector2D(1,1) )
			, _ButtonColorAndOpacity(FLinearColor::White)
			, _ForegroundColor( FCoreStyle::Get().GetSlateColor( "InvertedForeground" ) )
			, _IsFocusable( true ) { }

		...
		WidgetArgsType& OnPressed(const FSimpleDelegate& InDelegate) {
			_OnPressed = InDelegate;
			return *this;
		}

		WidgetArgsType& OnPressed(FSimpleDelegate&& InDelegate) {
			_OnPressed = MoveTemp(InDelegate);
			return *this;
		}
		...

		FSimpleDelegate _OnPressed;
	};
};
```
当然，实际上`SLATE_EVENT`当中所展开的函数远远不止这两个函数，碍于篇幅内容在此处就展开这么多，但其实这么一点内容我们也已经能得到有用的信息了。回忆一下在`UButton`处我们是怎么传入OnPressed事件的？没错，就是对对应FArgument调用`.OnPressed(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandlePressed))`，展开其调用也就是：
```cpp
SButton::FArgument::OnPressed(FSimpleDelegate::CreateUObject(this, &ThisClass::SlateHandlePressed));
```
也就是调用了`OnPressed`接收一个右值的重载，然后在这个函数中将传入的Delegate赋值给FArguments当中的成员变量，紧接着返回自身（建造者设计模式）方便继续构建。

好了，现在我们已经知道了这个用来传递给`SButton`用来初始化的参数是通过首先构建一个对应的`FArgument`，紧接着通过将这个构建好的Argument传递给对应的`void Construct(const FArguments& InArgs)`函数从而将输入事件的绑定传递到`SWidget`层级的了，简单的用一张图来表示就是：

<div align="center">
  <img src="./images/SWidget构建流程图.png">
</div>

### 输入事件从Slate传递到UMG
在上面，我们已经知道了UMG和Slate是如何关联并且传递所需要的输入事件委托的了，以Pressed事件为例，在`UButton`当中，传递给`SButton`的事件为——`SlateHandlePressed`，具体我们能看其定义：
```cpp
// In module Runtime/UMG/Conmponents, file Button.h
UCLASS()
class UMG_API UButton : public UContentWidget {
	...

	/** Called when the button is pressed */
	UPROPERTY(BlueprintAssignable, Category="Button|Event")
	FOnButtonPressedEvent OnPressed;

	...
};

// In module Runtime/UMG/Components, file Button.cpp
void UButton::SlateHandlePressed() {
	OnPressed.Broadcast();
}
```
实际上就是直接向绑定到OnPressed委托的所有事件进行一次广播，也就是我们在UMG蓝图中所绑定的、或者是在UMG C++中所绑定的按压事件。

### 以鼠标点击按钮为例，查看调用堆栈调用顺序
按照上面所分析的，我们可以先想象以下，在Windows平台下，一个鼠标的左键点击到一个按钮上，所绑定的OnPressed事件调用在引擎中的调用顺序会是怎么样的（从底往上）：
1. `FEngineLoop::Tick()`：每一帧调用。
2. `FSlateApplication::PollGameDeviceState()`：在每一帧中尝试获取当前输入设备的状态。
3. `FWindowsApplication::PollGameDeviceState(float TimeDelta)`：在`FSlateApplication`中调用到对应平台的`FWindowsApplication`的PollDevice函数。
4. 对应Device的`IInputDevice::SendControllerEvent()`：这个函数负责查询按键状态是否有改变，如果有，就发送输入通知，此处我们假设发出了鼠标左键按下的事件。
5. `FSlateApplication::OnMouseDown(const TSharedPtr<FGenericWindow>&, const EMouseButtons::Type)`：输入通知发送到`FSlateApplication`中对应的输入事件。
6. `FSlateApplication::OnMouseDown(const TSharedPtr<FGenericWindow>&, const EMouseButtons::Type, const FVector2D)`：上面的同名函数获取到鼠标位置后传入当前函数。
7. `FSlateApplication::ProcessMouseButtonDownEvent(const TSharedPtr<FGenericWindow>&, const FPointerEvent&)`：上面的同名函数处理手机屏幕触摸事件后，如果判断不是触摸事件，传入本函数。本函数负责按照一定的顺序遍历焦点路径中的每一个Slate，然后调用对应SlateWidget中的对应事件。
8. `SButton::OnMouseButtonDown(const FGeometry&, const FPointerEvent&)`：处理Press事件，并返回。
0. `SButton::Press()`：执行`OnPressed`委托所绑定的事件。
10. `UButton::SlateHandlePressed()`：这个函数为绑定到`SButton::OnPressed`当中的事件，负责广播所有绑定到`UButton::OnPressed`的事件存在一个鼠标左键输入。
11. 指定按钮绑定的函数。

无图无真相，就直接来看一下引擎当中处理UI输入的调用堆栈。
我们直接在引擎的`UButton`中处理OnPressed事件的函数中打一个断点
<div align="center">
  <img src="./images/OnPressedBreakpoint.png">
</div>

随便点击项目中的一个按钮，我们能看到到达`UButton::SlateHandlePressed`之前的调用堆栈：
<div align="center">
  <img src="./images/PressedCallStack.png">
</div>

可以看到，除开接收消息处理这一块，大体上和我们所设想的差不多——在每一帧中都会主动处理输入消息、然后将消息发送到`FSlateApplication`中对应的事件当中、再根据`FEventRouter::Route`函数来遍历焦点路径中的每一个SlateWidget、最后通过委托把输入事件通知到UWidget层级以调用我们所绑定的事件。

接下来我们来着重看一下不同的地方，也就是Windows平台是如何处理输入消息的：
在普通的Windows程序中，一般是在创建窗口的过程中绑定一个过程函数来处理输入的信息，整体类似于下面这样的：
```cpp
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam) {
    switch(uMsg) {
        case WM_KEYDOWN:
            if (wParam == VK_ESCAPE) PostQuitMessage(0);  // 按下ESC退出程序
            break;
        case WM_LBUTTONDOWN:
            // 处理鼠标左键点击坐标（通过lParam解析）
            break;
    }
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}
```
而在调用堆栈中，我们看到的`FWindowsApplication::AppWndProc`就是这么一个函数：
```cpp
LRESULT CALLBACK FWindowsApplication::AppWndProc(HWND hwnd, uint32 msg, WPARAM wParam, LPARAM lParam) {
	ensure( IsInGameThread() );
	return WindowsApplication->ProcessMessage( hwnd, msg, wParam, lParam );
}
```
能看到接下来就是在堆栈中顺着往下走的`FWindowsApplication::ProcessMessage`函数了。我们能在里面找到一个超级长的switch代码段，下面就列下我们所关注的部分：
```cpp
int32 FWindowsApplication::ProcessMessage( HWND hwnd, uint32 msg, WPARAM wParam, LPARAM lParam ) {
	...
	switch(msg) {
		...
		case WM_KEYDOWN:
		case WM_SYSKEYUP:
		case WM_KEYUP:
		case WM_LBUTTONDBLCLK:
		case WM_LBUTTONDOWN:
		case WM_MBUTTONDBLCLK:
		case WM_MBUTTONDOWN:
		case WM_RBUTTONDBLCLK:
		case WM_RBUTTONDOWN:
		case WM_XBUTTONDBLCLK:
		case WM_XBUTTONDOWN:
		case WM_XBUTTONUP:
		case WM_LBUTTONUP:
		case WM_MBUTTONUP:
		case WM_RBUTTONUP:
		case WM_NCMOUSEMOVE:
		case WM_MOUSEMOVE:
		case WM_MOUSEWHEEL:
#if WINVER >= 0x0601
		case WM_TOUCH:
#endif
			{
				DeferMessage( CurrentNativeEventWindowPtr, hwnd, msg, wParam, lParam );
				// Handled
				return 0;
			}
			break;

		case WM_SETCURSOR:
			...
	};
}
```
这部分代码表明，对于比如说是按钮按下、鼠标移动、鼠标按键按下等等的这一系列输入事件都会调用到`DeferMessage`函数当中进行处理，而在`DeferMessage`实际上也就是调用到`ProcessDeferredMessage`当中。那现在，我们就只需要关注这个函数即可。
与`ProcessMessage`函数类似，`ProcessDeferredMessage`当中同样有一个很大的switch代码块来对不同类型的输入进行不同的处理，包括`WM_KEYDOWN`、`WM_KEYUP`等，而我们这个例子中是通过鼠标的点击来触发的，因此我们只需要关注`WM_LBUTTONDOWN`：
```cpp
int32 FWindowsApplication::ProcessDeferredMessage( const FDeferredWindowsMessage& DeferredMessage ) {
	switch(msg) {
		...
		// Mouse Button Down
		case WM_LBUTTONDBLCLK:
		case WM_LBUTTONDOWN:
		case WM_MBUTTONDBLCLK:
		case WM_MBUTTONDOWN:
		case WM_RBUTTONDBLCLK:
		case WM_RBUTTONDOWN:
		case WM_XBUTTONDBLCLK:
		case WM_XBUTTONDOWN:
		case WM_LBUTTONUP:
		case WM_MBUTTONUP:
		case WM_RBUTTONUP:
		case WM_XBUTTONUP:
			{
				// 获取鼠标位置
				POINT CursorPoint;
				CursorPoint.x = GET_X_LPARAM(lParam);
				CursorPoint.y = GET_Y_LPARAM(lParam); 

				ClientToScreen(hwnd, &CursorPoint);

				const FVector2D CursorPos(CursorPoint.x, CursorPoint.y);

				// 初始化参数
				EMouseButtons::Type MouseButton = EMouseButtons::Invalid;
				bool bDoubleClick = false;
				bool bMouseUp = false;

				// 根据鼠标事件再次划分赋值
				switch(msg) {
				...
				case WM_LBUTTONDOWN:
					MouseButton = EMouseButtons::Left;
					break;
				};
				...

				// 最后，根据收集到的信息，向MessageHandler发送对应的输入事件
				if (bMouseUp) {
					return MessageHandler->OnMouseUp( MouseButton, CursorPos ) ? 0 : 1;
				} else if (bDoubleClick) {
					MessageHandler->OnMouseDoubleClick( CurrentNativeEventWindowPtr, MouseButton, CursorPos );
				} else {
					MessageHandler->OnMouseDown( CurrentNativeEventWindowPtr, MouseButton, CursorPos );
				}
				return 0;
			}
		...
	}
}
```
在之前，我们已经分析过了，MessageHandler只能是当前程序运行过程中的SlateApplication实例，也就是回到我们之前分析所得之的部分了。

### 总结
用一幅图总结一下，从按键输入到UI响应，总共需要以下的一些步骤：
<div align="center">
  <img src="./images/输入处理流程.jpg">
</div>