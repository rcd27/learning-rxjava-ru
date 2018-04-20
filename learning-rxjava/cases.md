Here are some examples by [Stephen D'Amico at DroidCon 2017](https://www.youtube.com/watch?v=q4eK3VFhnA0&feature=youtu.be)

#### Retrying requests
- `retryWhen()`
```java
PublishRelay<Long> retryRequest = PublishRelay.create();

getRequestObservable()
  .retryWhen(attempt -> retryRequest)
  .subscribe(viewModel -> ..)
 
@OnClick(R.id.retry_button)
public void onRetryClicked() {
  retryRequest.call(System.currentTimeMillis);
}
```

#### Using `Response<T>` with `share()`
```java
interface Api {
  @GET("events")
  Observable<Response<EventsResponse>> getEvents();
}

Observable<Response<EventsResponse>> eventsResponse =
api.getEvents()
  .observeOn(AndroidSchedulers.mainThread())
  .share();
  
eventsResponse
  .filter(r -> r.code == HTTP_403)
  .subscribe(this::handle403Response);
  
eventsResponse
  .filter(r -> r.code == HTTP_401)
  .subscribe(this::handle401Response);
  
eventsResponse
  .filter(r -> r.code == HTTP_500)
  .subscribe(this::handle500Response);
```

// TODO: checkout ContentLoadingProgressBar

#### Showing loading (and the state in general)
```java
enum RequestState {
  IDLE, LOADING, COMPLETE, ERROR
}

// Можно также использовать Subject
// Создаём мост между императивным программированием и эрыксом
BehaviourRelay<RequestState> state = 
BehaviourRelay.create(RequestState.IDLE);

// Здесь обрабатываем ответы от сервера
Observable.just(trigger)
  .doOnNext(() -> state.call(RequestState.LOADING))
  .observeOn(Schedulers.io())
  .flatMap(trigger -> api.getEvents())
  .observeOn(AndroidSchedulers.mainThread())
  .doOnError(t -> state.call(RequestState.ERROR))
  .doOnComplete(() -> state.call(RequestState.COMPLETE))
  .subscribe(events -> .. , error -> ..);
  
// А вот тут вьюха подписывается собственно на стэйт
state.subscribe(requestState -> {
                switch(requestState) {
                  IDLE:
                      break;
                  LOADING:
                      loadingIndicator.show();
                      errorView.hide();
                      break;
                  COMPLETE:
                      loadingIndicator.hide();
                      break;
                  ERROR:
                      ..
                      break;
                }
            });
```

- Разносим UI и запросы в сеть ещё дальше:
```java
class ApiRequest {
BehaviorRelay<RequestState> state = 
  BehaviorRelay.create(RequestState.IDLE);
BehaviorRelay<EventsResponse> response = BehaviorRelay.create();
BehaviorRelay<Throwable> errors = BehaviorRelay.create();
BehaviorRelay<Long> trigger = BehaviorRelay.create();

public ApiRequest() {
    trigger
      .doOnNext(() -> state.call(RequestState.LOADING))
      .observeOn(Schedulers.io())
      .flatMap(trigger -> api.getEvents())
      .doOnError(t -> state.call(RequestState.ERROR))
      .doOnError(errors)
      .onErrorResumeNext(Observable.empty())
      .doOnNext(() -> state.call(RequestState.COMPLETE))
      .subscribe(response);
  }

  void execute() {
    trigger.call(System.currentTimeMillis());
  }
}

// где-то ближе к UI слою
ApiRequest request = new ApiRequest(api.getEvents());

request.state.subscribe(state -> updateLoadingView(state));
request.state.subscribe(state -> updateLoadingNotification(state));
request.response.subscribe(eventsResponse -> showEvents(eventsResponse))
request.errors.subscribe(t -> showError(t));

request.execute();
```



