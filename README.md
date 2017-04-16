# ReactorKit

ReactorKit is a framework for reactive and unidirectional Swift application architecture. This repository introduces the basic concept of ReactorKit and describes how to build an application using ReactorKit.

You may want to see [Examples](#examples) section first if you'd like to see the actual code.

---

## ⚠️ Prereleasing Stage

ReactorKit is currently in prereleasing stage. Everything can be changed in the future. Major changes can be found in the [Changelog](#changelog) section.

---

## Table of Contents

* [Basic Concept](#basic-concept)
* [Layers](#layers)
    * [View](#view)
    * [ViewReactor](#viewreactor)
    * [Model](#model)
    * [Service](#service)
    * [ServiceProvider](#serviceprovider)
* [Conventions](#conventions)
* [Advanced Usage](#advanced-usage)
    * [Presenting next ViewController](#presenting-next-viewcontroller)
    * [Communicating between ViewReactor and ViewReactor](#communicating-between-viewreactor-and-viewreactor)
* [Examples](#examples)
* [Changelog](#changelog)
* [License](#license)

## Basic Concept

ReactorKit is a combination of [Flux](https://facebook.github.io/flux/) and [Reactive Programming](https://en.wikipedia.org/wiki/Reactive_programming). The user actions and the view states are delivered to each layer via observable streams. These streams are unidirectional so the view can only emit actions and the reactor can only emit states.

<p align="center">
  <img alt="flow" src="https://cloud.githubusercontent.com/assets/931655/25073432/a91c1688-2321-11e7-8f04-bf91031a09dd.png" width="600">
</p>

## Layers

### View

*View* displays data. A view controller and a cell are treated as a view. The view binds user-inputs to the reactor's action stream and the states to each UI components.

```swift
func bind(reactor: Reactor) {
  // action
  refreshButton.rx.tap.map { Reactor.Action.refresh }
    .bindTo(reactor.action)
    .addDisposableTo(self.disposeBag)

  // state
  reactor.state.map { $0.isFollowing }
    .bindTo(followButton.rx.isSelected)
    .addDisposableTo(self.disposeBag)
}
```

### Reactor

*Reactor* is an UI independent layer which receives user inputs and creates output. The ViewReactor has two types of property: *Input* and *Output*. The Input property represents the exact user input occured in the View. The Input property is formed like `loginButtonDidTap` rather than `login()`. The Output property provides the primitive data so that the View can bind it to the UI components without converting values.

ViewReactor **must not** have the reference of the View instance. However, in order to provide primitive data, ViewReactor knows the indirect information about which values the View needs.

### Service

Reactive Architecture has a special layer named *Service*. Service layer does actual business logic such as networking. The ViewReactor is a middle layer which manages event streams. When the ViewReactor receives user inputs from the View, the ViewReactor manipulates and compose the event streams and passes them to the Service. Then the Service makes a network request, maps the response to the Model, and send it back to the ViewReactor.

<p align="center">
  <img alt="service-layer" src="https://cloud.githubusercontent.com/assets/931655/24015672/7b4d8916-0acc-11e7-8f6c-0cee4a6d8f0e.png" width="600">
</p>

### ServiceProvider

A single ViewReactor can communicate with many Services. *ServiceProvider* provides the references of the Services to the ViewReactor. The ServiceProvider is created once in the whole application life cycle and passed to the first ViewReactor. The first ViewReactor should pass the same reference of the ServiceProvider instance to the child ViewReactor.

## Conventions

Reactive Architecture suggests some conventions to write clean and concise code.

* You should use `PublishSubject` for Input properties and `Driver` for Output properties.

    ```swift
    protocol MyViewReactorType {
      // Input
      var loginButtonDidTap: PublishSubject<Void> { get }

      // Output
      var isLoginButtonEnabled: Driver<Bool> { get } 
    }
    ```

* ViewReactor should have the ServiceProvider as the initializer's first argument.

    ```swift
    class MyViewReactor {
      init(provider: ServiceProviderType)
    }
    ```

* You must create a ViewReactor outside of the View. Pass the ViewReactor to the initializer if the View is not reusable. Pass the ViewReactor to the `configure(reactor:)` method if the View is reusable.

    **Bad**

    ```swift
    class MyViewController {
      let reactor = MyViewReactor()
    }
    ```

    **Good**

    ```swift
    let reactor = MyViewReactor(provider: provider)
    let viewController = MyViewController(reactor: reactor)
    ```

* The ServiceProvider should be created and passed to the first-most View.

    ```swift
    let serviceProvider = ServiceProvider()
    let firstViewReactor = FirstViewReactor(provider: serviceProvider)
    window.rootViewController = FirstViewController(reactor: firstViewReactor)
    ```

* The View should not have control flow. It means that the View cannot modify the data. The View only knows how to map the data.

    **Bad**

    ```swift
    reactor.titleLabelText
      .map { $0 + "!" } // Bad: View should not modify the data
      .bindTo(self.titleLabel)
    ```

    **Good**

    ```swift
    reactor.titleLabelText
      .bindTo(self.titleLabel.rx.text)
    ```

* The View should not know what the ViewReactor does. The View can only communicate to ViewReactor about what the View did.

    **Bad**

    ```swift
    protocol MyViewReactorType {
      // Bad: View knows what the ViewReactor does (login)
      var login: PublishSubject<Void> { get }
    }
    ```

    **Good**

    ```swift
    protocol MyViewReactorType {
      // View just says "Hey I clicked the login button"
      var loginButtonDidTap: PublishSubject<Void> { get }
    }
    ```

* The ViewReactor should hide the Model. The ViewReactor only exposes the minimum data so that the View can render.

    **Bad**

    ```swift
    struct ProductViewReactor {
      let product: Driver<Product> // Bad: ViewReactor should hide Model
    }
    ```

    **Good**

    ```swift
    struct ProductViewReactor {
      let productName: Driver<String>
      let formattedPrice: Driver<String>
      let formattedOriginalPrice: Driver<String>
      let isOriginalPriceHidden: Driver<Bool>
    }
    ```

* You should use protocols to have loose dependency. Usually the ViewReactor, Service and ServiceProvider have its corresponding protocols.

    ```swift
    protocol ServiceProviderType {
      var userDefaultsService: UserDefaultServiceType { get }
      var keychainService: KeychainServiceType { get }
      var authService: AuthServiceType { get }
      var userService: UserServiceType { get }
    }
    ```

    ```swift
    protocol UserServiceType {
      func user(id: Int) -> Observable<User>
      func updateUser(id: Int, name: String?) -> Observable<User>
      func followUser(id: Int) -> Observable<Void>
    }
    ```

## Advanced Usage

This chapter describes some architectural considerations for real world.

### Presenting next ViewController

Almost applications have more than one ViewController.  In MVC architecture, ViewController(`ListViewController`) creates next ViewController(`DetailViewController`) and just presents it. This is same in Reactive Architecture but the only difference is the creation of ViewReactor.

In Reactive Architecture, `ListViewReactor` creates `DetailViewReactor` and passes it to `ListViewController`. Then the `ListViewController` creates `DetailViewController` with the `DetailViewReactor` received from `ListViewReactor`.

* **ListViewReactor**

    ```swift
    class ListViewReactor: ListViewReactorType {
      // MARK: Input
      let detailButtonDidTap: PublishSubject<Void> = .init()

      // MARK: Output
      let presentDetailViewReactor: Observable<DetailViewReactorType> // No Driver here

      // MARK: Init
      init(provider: ServiceProviderType) {
        self.presentDetailViewReactor = self.detailButtonDidTap
          .map { _ -> DetailViewReactorType in
            return DetailViewReactor(provider: provider)
          }
      }
    }
    ```

* **ListViewController**

    ```swift
    class ListViewController: UIViewController {
      private func configure(reactor: ListViewReactorType) {
        // Output
        reactor.detailViewReactor
          .subscribe(onNext: { reactor in
            let detailViewController = DetailViewController(reactor: reactor)
            self.navigationController?.pushViewController(detailViewController, animated: true)
          })
          .addDisposableTo(self.disposeBag)
      }
    }
    ```

### Communicating between ViewReactor and ViewReactor

Sometimes ViewReactor should receive data (such as user input) from the other ViewReactor. In this case, use `rx` extension to communicate between View and View. Then bind it to ViewReactor.
    
<p align="center">
  <img alt="viewreactor-viewreactor" src="https://cloud.githubusercontent.com/assets/931655/24015677/7d08d8be-0acc-11e7-9fc2-78057f87cda0.png" width="600">
</p>

* **MessageInputView.swift**

    ```swift
    extension Reactive where Base: MessageInputView {
      var sendButtonTap: ControlEvent<String?> { ... }
      var isSendButtonLoading: ControlEvent<String?> { ... }
    }
    ```

* **MessageListViewReactor.swift**

    ```swift
    protocol MessageListViewReactorType {
      // Input
      var messageInputViewSendButtonDidTap: PublishSubject<String?> { get }

      // Output
      var isMessageInputViewSendButtonLoading: Driver<Bool> { get }
    }
    ```

* **MessageListViewController.swift**

    ```swift
    func configure(reactor: MessageListViewReactorType) {
      // Input
      self.messageInputView.rx.sendButtonTap
        .bindTo(reactor.messageInputViewSendButtonDidTap)
        .addDisposableTo(self.disposeBag)

      // Output
      reactor.isMessageInputViewSendButtonLoading
        .drive(self.messageInputView.rx.isSendButtonLoading)
        .addDisposableTo(self.disposeBag)
    }
    ```

## Examples

* [RxTodo](https://github.com/devxoul/RxTodo): iOS Todo Application using Reactive Architecture
* [Cleverbot](https://github.com/devxoul/Cleverbot): Cleverbot for iOS using Reactive Architecture
* [Drrrible](https://github.com/devxoul/Drrrible): Dribbble for iOS using Reactive Architecture

## Changelog

* 2017-03-17
    * Change the architecture name from RxMVVM to The Reactive Architecture.
    * Every ViewModels are renamed to ViewReactors.

## License

[Creative Commons Attribution 4.0 International license](http://creativecommons.org/licenses/by/4.0/)
