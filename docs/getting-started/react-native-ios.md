---
id: react-native-ios
title: Manually set up your React Native iOS App
sidebar_label: React Native for iOS
---

These instructions are aimed at people manually adding Flipper to a React Native 0.62+ app.
This should only be necessary if you have an existing app that cannot be upgraded with the
[React Native Upgrade tool](https://reactnative.dev/docs/upgrading).

## Dependencies

Add this code to your `ios/Podfile`:

```ruby
platform :ios, '9.0'

def flipper_pods()
  flipperkit_version = '0.37.0'
  pod 'FlipperKit', '~>' + flipperkit_version, :configuration => 'Debug'
  pod 'FlipperKit/FlipperKitLayoutPlugin', '~>' + flipperkit_version, :configuration => 'Debug'
  pod 'FlipperKit/SKIOSNetworkPlugin', '~>' + flipperkit_version, :configuration => 'Debug'
  pod 'FlipperKit/FlipperKitUserDefaultsPlugin', '~>' + flipperkit_version, :configuration => 'Debug'
  pod 'FlipperKit/FlipperKitReactPlugin', '~>' + flipperkit_version, :configuration => 'Debug'
end

# Post Install processing for Flipper
def flipper_post_install(installer)
  installer.pods_project.targets.each do |target|
    if target.name == 'YogaKit'
      target.build_configurations.each do |config|
        config.build_settings['SWIFT_VERSION'] = '4.1'
      end
    end
  end
  file_name = Dir.glob("*.xcodeproj")[0]
  app_project = Xcodeproj::Project.open(file_name)
  app_project.native_targets.each do |target|
    target.build_configurations.each do |config|
      cflags = config.build_settings['OTHER_CFLAGS'] || '$(inherited) '
      unless cflags.include? '-DFB_SONARKIT_ENABLED=1'
        puts 'Adding -DFB_SONARKIT_ENABLED=1 in OTHER_CFLAGS...'
        cflags << '-DFB_SONARKIT_ENABLED=1'
      end
      config.build_settings['OTHER_CFLAGS'] = cflags
    end
    app_project.save
  end
  installer.pods_project.save
end

target 'your-app-name' do
  ...


  # Replace the existing yoga import with the following (adding modular_headers):
  pod 'Yoga', :path => '../node_modules/react-native/ReactCommon/yoga', :modular_headers => true

  ...
  use_native_modules!

  # For enabling Flipper.
  # Note that if you use_framework!, flipper will not work.
  # Disable these lines if you are doing use_framework!
  flipper_pods()
  post_install do |installer|
    flipper_post_install(installer)
  end
end
```

Install the dependencies by running `cd ios && pod install`. You can now
import and initialize Flipper in your AppDelegate.

## Initialization

The code below enables the following integrations:

- Layout Inspector
- Network
- Shared Preferences
- Crash Reporter

<!--DOCUSAURUS_CODE_TABS-->

<!--Objective-C-->

```objective-c
...
#if DEBUG
#ifdef FB_SONARKIT_ENABLED
#import <FlipperKit/FlipperClient.h>
#import <FlipperKitLayoutPlugin/FlipperKitLayoutPlugin.h>
#import <FlipperKitLayoutPlugin/SKDescriptorMapper.h>
#import <FlipperKitNetworkPlugin/FlipperKitNetworkPlugin.h>
#import <FlipperKitReactPlugin/FlipperKitReactPlugin.h>
#import <FlipperKitUserDefaultsPlugin/FKUserDefaultsPlugin.h>
#import <SKIOSNetworkPlugin/SKIOSNetworkAdapter.h>
#endif
#endif

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  [self initializeFlipper:application];
  ...
}

- (void) initializeFlipper:(UIApplication *)application {
  #if DEBUG
  #ifdef FB_SONARKIT_ENABLED
    FlipperClient *client = [FlipperClient sharedClient];
    SKDescriptorMapper *layoutDescriptorMapper = [[SKDescriptorMapper alloc] initWithDefaults];
    [client addPlugin: [[FlipperKitLayoutPlugin alloc] initWithRootNode: application withDescriptorMapper: layoutDescriptorMapper]];
    [client addPlugin: [[FKUserDefaultsPlugin alloc] initWithSuiteName:nil]];
    [client addPlugin: [FlipperKitReactPlugin new]];
    [client addPlugin: [[FlipperKitNetworkPlugin alloc] initWithNetworkAdapter:[SKIOSNetworkAdapter new]]];
    [client start];
  #endif
  #endif
}

@end
```

<!--Swift-->

```swift
...
#if DEBUG
#if FB_SONARKIT_ENABLED
import FlipperKit
#endif
#endif

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

  func application(
      _ application: UIApplication,
      didFinishLaunchingWithOptions
      launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
    initializeFlipper(with: application)
    ...
  }

  private func initializeFlipper(with application: UIApplication) {
    #if DEBUG
    #if FB_SONARKIT_ENABLED
      let client = FlipperClient.shared()
      let layoutDescriptorMapper = SKDescriptorMapper(defaults: ())
      FlipperKitLayoutComponentKitSupport.setUpWith(layoutDescriptorMapper)
      client?.add(FlipperKitLayoutPlugin(rootNode: application, with: layoutDescriptorMapper!))
      client?.add(FKUserDefaultsPlugin(suiteName: nil))
      client?.add(FlipperKitReactPlugin())
      client?.add(FlipperKitNetworkPlugin(networkAdapter: SKIOSNetworkAdapter()))
      client?.add(FlipperReactPerformancePlugin.sharedInstance())
      client?.start()
    #endif
    #endif
  }
}
```

<!--END_DOCUSAURUS_CODE_TABS-->

Lastly, open the Flipper desktop app, and run `yarn ios` in your terminal.

## Troubleshooting

See the [troubleshooting page](../troubleshooting) for help with known problems.

## Further Steps

To create your own plugins and integrate with Flipper using JavaScript, check out our [writing plugins for React Native](tutorial/react-native) tutorial!
