# Blackmagic Switcher SDK Bug

This example is based off the `DeviceInfo` sample project. It will only run on macOS, however the code could easily be adapted to run on the Windows SDK.

At startup we register two `MyCallbackClass` like the following:

```cpp
MyCallbackClass* b = new MyCallbackClass(&switcher, nil);
MyCallbackClass* a = new MyCallbackClass(&switcher, b);
```

You'll notice how the *first* callback has a reference to the *second* callback. This will become important later on.

We then register them to the `switcher` like the following:

```cpp
// Notice the order is A -> B, while the definitions earlier were the other way around.
// It's important that A runs first and this seems to dictate that.
switcher->AddCallback(a);
switcher->AddCallback(b);
```

Now in the `MyCallbackClass::Notify` method we check for the argument and if it's not `nil` we remove that callback.

We only do this on the disconnect event but this is just to avoid too much console output.

```cpp
HRESULT MyCallbackClass::Notify(BMDSwitcherEventType eventType, BMDSwitcherVideoMode coreVideoMode)
{
    // Filtering down to the event doesn't change the bug but it makes it clearer what's going on.
    if (eventType == bmdSwitcherEventTypeDisconnected) {
        std::cout << "\n" << this << " DISCONNECTED! " << std::endl;

        if (other != nil) {
            HRESULT result = switcher->get()->RemoveCallback(other);
            std::cout << this << " removing callback " << other << " " << result << std::endl;
            if (result != S_OK) {
                std::cout << "ERROR REMOVING CALLBACK!!!!!!" << std::endl;
            }
        }
    }

    return S_OK;
}
```

When we run this code you'll see something like the following:

```cpp
Switcher found at 10.0.0.205
 Product Name:                            ATEM Mini Pro

0x600003daa420 DISCONNECTED!
0x600003daa420 removing callback 0x600003daa400 0

0x600003daa400 DISCONNECTED!
```

Notice how it says "removing callback 0x600003daa400" but then we receive the "0x600003daa400 DISCONNECTED!" later on.

We called `RemoveCallback` and the SDK returned `S_OK` but the callback *clearly* hasn't been removed and is invoked.

In my specific usecase when `RemoveCallback` reported being successful we dropped any memory associated with the callback and so it firing causing the application to access freed memory and the application crashes with a segfault.
