## 2중 Observable 

가령 예를 들어 이러한 상황이 존재한다.

특정 버튼을 Tab 하면 -> Logic 을 통해 성공 / 실패 를 방출하고,
성공/방출 즉, 결과에 따라 그 뒤에 액션을 다르게 해줘야 하는 경우를 생각할수있다. 
좀 더 구체적인 예시를 들면, 로그인 화면을 생각해 볼 수 있겠다.

우리는 로그인을 하기 위해서는 대체적으로 ID + PW 를 입력해서 로그인을 하게된다.
그러나 로그인에서 실패하는 API의 Response를 받아오면 실패했다는것을 유저에게 Noti해줘야할 의무가있고,
만약 성공했다면 로그인 성공해서 다른 컨텐츠들을 즐길수 있게 도와줘야 한다는 소리다.

결국은 이러한 로직은 다음과 같다.

- ID/PW 입력 -> 버튼 Tab -> API Action (API Observable 생성) -> API 결과 토대로 ViewModel 에서 해당 이벤트를 구독
- 구독한 스트리밍의 이벤트 결과를 토대로 이벤트를 나눠야 함.

로직 자체는 굉장히 간단하지만, 이벤트를 나누는 과정을 어떻게 해야할지 고민이 좀 많았다.
나같은경우는 API 로 생성한 Result<SignInResponse, NetworkErr>로 스트리밍을 따로 생성하고,
실패시 Alert를 방출해줄 Signal과, 성공시 Void를 drive해줄 Driver로 구독해주고 보내주는 이벤트를 filtering하고 매핑하여 반환하는 과정으로 이 문제를 해결했다.

코드로 표현하면 ViewModel의 코드는 다음과 같다.
```swift
// at SignInViewModel.swift

let requestData = requestResult
            .compactMap { data -> SignInAction in
                switch data {
                case let .success(result):
                    guard let token = result.token else { return .nonExist }
                    KeyChain.shared.addItem(key: "token", value: token.token)
                    KeyChain.shared.addItem(key: "token", value: token.refreshToken)
                    print("SUCCESSTOKEN")
                    return .success
                case let .failure(e) :
                    print(e.localizedDescription)
                    return .networkErr
                default:
                    return .networkErr
                }
            }
            .asObservable()
            //.asDriver(onErrorDriveWith: .empty())
let t = requestData.subscribe{
            print($0)
        }.disposed(by: disposeBag)
        
self.errorCheck = requestData
            .filter { $0 != .success }
            .map { action in
                return model.setAlert(action: action)
            }
            .asSignal(onErrorSignalWith: .empty())
        
self.nextCheck = requestData
            .filter { $0 == .success }
            .map { action -> Void in
                return Void()
            }
            .asDriver(onErrorDriveWith: .empty())
        
```

```swift
// at SignInViewController(View)

submitButton.rx.tap
            .bind(to: viewModel.submitButtonTapped)
            .disposed(by: disposeBag)
        
viewModel.nextCheck
            .drive(onNext: {
                self.navigationController?.popViewController(animated: true)
            })
            .disposed(by: disposeBag)
        
viewModel.errorCheck
            .emit(to: self.rx.setAlert)
            .disposed(by: disposeBag)
```
