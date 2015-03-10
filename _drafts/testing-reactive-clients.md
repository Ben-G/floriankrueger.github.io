---
layout:             post
title:              "Testing Reactive Clients"
#date:              XXXX-XX-XX XX:XX:XX
categories:         iOS OSX Cocoa ReactiveCocoa Testing
cover:              '/assets/images/post-testing-reactive-clients/cover.png'
cover-author-name:  "MADE IN MOMENTS"
cover-author-url:   "http://www.madeinmoments.com"
published:          false
sitemap:
  lastmod:          2015-03-09
  priority:         0.7
  changefreq:       'monthly'
---

Unit testing is a controverse topic in the iOS Developer Community. We all know we should write tests, but often we don't live up to this ideal. One of the reasons is that the tools we use didn't support writing and executing unit tests for a long time and setting them up was kind of a pain.

In recent years this has changed, but we all got used to a certain speed when it comes to developing iOS applications. To be honest: writing tests makes you slower - at least in the beginning. Most often we are not able or not willing to sacrifice our speed in order to learn how to efficiently write unit tests by just doing it in our everyday work. And looking back at a couple of years that were possible without tests seem to prove that *"we are good enough to write applications without tests"*.

I'm also guilty of using these excuses myself although I'm very well aware that I really should write more tests. So I try to do it every now and then but when things get somewhat intense and deadlines come closer .. you know the situation.

Anyways - last summer I saw the excellent talks by John Reid and Graham Lee and heard about how they do unit tests as well as end-to-end tests at facebook. I started thinking about the topic more again and realized how reassuring it must be to have a big test coverage in a project. Especially in project that are somewhat bigger than your average todo list app. And being able to develop everything - even your network code - when you're offline is a big big bonus (given that you have a correct and complete documentation on your hands).

So on my way back from NSSpain I started applying those methods to my current tasks. John Reid demonstrated how to mock network communication build upon AFNetworking - without any mocking framework. I never thought of building the mocking part of my tests myself but the example looked so simple and pragmatic that I wanted to try it. When starting to design tests I ran into the following problem: our HTTP client is 100% reactive - built upon *ReactiveCocoa* and *AFNetworking-RACExtensions*. The given examples in the talk, which involved asynchronous calls, store the callback blocks and execute them with mocked objects. In RAC there are no blocks to store and call (at least not at the level relevant to the tests), so I needed to work around this. The solution isn't too complex but it took me a while to figure out how to do it efficiently and without too much boilerplate code so here's what I did.

I don't claim that this is *the* solution. It's just what I came up with so use it on your own risk and send me any feedback you have, good or bad.

## Faking the HTTP Client

The key of the unit testing method John demonstrated is faking the HTTP connection at the point where the client returns the parsed JSON to the caller. The client under consideration calls the `- (RACSignal *)rac_GET:(NSString *)path parameters:(NSDictionary *)parameters` method from *AFNetworking-RACExtensions* Pod. I was lucky as these calls were already encapsulated.

```objective-c
#pragma mark - HTTP Methods

- (RACSignal *)GET:(NSString *)urlString parameters:(NSDictionary *)parameters
{
  return [self.httpClient rac_GET:urlString parameters:parameters];
}

- (RACSignal *)POST:(NSString *)urlString parameters:(NSDictionary *)parameters
{
  return [self.httpClient rac_POST:urlString parameters:parameters];
}

- (RACSignal *)PUT:(NSString *)urlString parameters:(NSDictionary *)parameters
{
  return [self.httpClient rac_PUT:urlString parameters:parameters];
}

// .. and so on
```

This is the point where my fake client could jump in and take over. So I created a Subclass of my client (let's call the client class `MYReactiveClient`) which I called `MYReactiveTestClient`. This client overrides the methods you can see above. I'm going to focus on a single HTTP Method from now on - GET - as it is almost identical for all the other methods. I'm sure you'll figure out what to do.

```objective-c
- (RACSignal *)GET:(NSString *)urlString parameters:(NSDictionary *)parameters
{
  self.lastUrlString = urlString;   // 1
  self.lastParameters = parameters; // 2
  self.getCallCount += 1;           // 3

  return self.subject;              // 4
}
```

As you can see, the fake client does some capturing before it returns a property of its own called `subject`. It first stores the called URL String (line 1) and the given parameters (line 2) and increases the count of GET calls (line 3). This is only for checking the correctness of the call later on in case it is interesting for your unit test.

The `subject` that is returned from this method (line 4) is a `RACSubject` that is given to the fake client before any call is made that would cause the client to call the GET method. I'm reusing this property for all HTTP Methods by the way as other assertions will be wrong anyways should the wrong method be called.

Another thing I needed to do in order for the client to work is the class method which normally returns the singleton instance of the client. The code under test uses this method to retrieve a client instance. For the test, it returns a new instance of the fake client for each call instead of a singleton, nothing magical here.

## Testing

So enough preface, let's write a unit test which uses the fake client.

```objective-c
- (void)testFetchLatestItems
{
  MYReactiveTestClient *client = [MYReactiveTestClient sharedInstance]; // 1
  client.subject = [RACSubject subject];                                // 2

  __block id val = nil;                                                 // 3

  [[client fetchLatestItems] subscribeNext:^(id x) {                    // 4
      val = x;
  }];

  NSDictionary *json = [MYResponseHelper randomLatestItemsResponse];    // 5
  [self.client.subject sendNext:json];                                  // 6
  [self.client.subject sendCompleted];                                  // 7

  XCTAssertTrue([val isKindOfClass:[MYLatestItemsResponse class]]);     // 8
  XCTAssertEqual(self.client.getCallCount, 1);                          // 9
  XCTAssertEqual([self.client totalCallCount], 1);                      // 10
}
```

This test does the following: It creates a new fake client in line 1 and initializes a new `RACSubject` which is assigned to the fake clients `subject` property in line 2. The `RACSubject` will later be used to simulate a successful network call that returns a parsed JSON object in form of a `NSDictionary`. In line 3 a block variable is declared which is used to store the value returned to the 'next' block of the signal. Line 4 calls the method under test (`fetchLatestItems`) and subscribes to the 'next' event. Within the 'next' block we're now storing the returned value inside `val`. After subscribing to the signal of the fake client, we're now able to send some data to the `RACSubject` to start the test: we create a `NSDictionary` in line 5, send a 'next' event with this dictionary to the subject (line 6) and then complete the subject (line 7).

After all the internal mechanics of the client have done their job we can assert three conditions to ensure that everything worked correctly. First (line 8), the `val` variable must be of class `MYLatestItemsResponse`. If it was `nil` the 'next' block wasn't called and there potentially an error (most likely the model doesn't match) or a problem with the client itself. Next, we need to know which networking method was called. In line 9 we make sure that the GET method was called exactly one time and in line 10 we make sure that only one method was called in total to be 200% sure that only the GET method was called.

## Reducing Boilerplate

There are a few things here that we can improve in order to reduce boilerplate. The first thing is to make the client and its setup part of the testclass. It is likely that we need the same client for all the tests in this suite so why not make use of `setUp` and `tearDown`? [using `sharedInstance` here might be semantically confusing for other team members (people might think you are not using a fresh client), possible to use other initializer?]

```objective-c
- (void)setUp
{
  [super setUp];
  self.client = [MYReactiveTestClient sharedInstance];
  self.client.subject = [RACSubject subject];
}

- (void)tearDown
{
  self.client = nil;
  [super tearDown];
}
```

We will need a property named `client` of type `MYReactiveTestClient` in order
for this to work of course.

With this modification in place, we can reduce the test a little in term of LOC:

```objective-c
- (void)testFetchLatestItems
{
  __block id val = nil;

  [[self.client fetchLatestItems] subscribeNext:^(id x) {
    val = x;
  }];

  NSDictionary *json = [MYResponseHelper randomLatestItemsResponse];
  [self.client.subject sendNext:json];
  [self.client.subject sendCompleted];

  XCTAssertTrue([val isKindOfClass:[MYLatestItemsResponse class]]);
  XCTAssertEqual(self.client.getCallCount, 1);
  XCTAssertEqual([self.client totalCallCount], 1);
}
```

The next thing one *could* do, is to give the client not a `RACSubject` but a concrete value to return (let's call the property `networkResult`). It could then return that value like this:

```objective-c
return [RACSignal return:self.networkResult];
```

With this in place you would be able to write your unit test a little more concise using the testing method provided by ReativeCocoa.

```objective-c
- (void)testFetchLatestItems
{
  self.client.networkResult = [MYResponseHelper randomLatestItemsResponse];

  id val = [[self.client fetchLatestItems] asynchronousFirstOrDefault:nil
                                                              success:NULL
                                                                error:NULL];

  XCTAssertTrue([val isKindOfClass:[MYLatestItemsResponse class]]);
  XCTAssertEqual(self.client.getCallCount, 1);
  XCTAssertEqual([self.client totalCallCount], 1);
}
```

Which one of the things you like more is a question of taste. With the first approach you have full control on what the `RACSubject` does in your test and a very dumb fake client. That comes of course with the cost of more code inside the test.

The second approach provides much shorter tests but some knowledge about the test to be run and complexity inside the client. This does only apply if you're about to test more than the success scenario of course, otherwise you're fine with the last approach shown above. But as we could and should also test the error scenarios we need to find a way to return the correct result from the fake client. Of course, there is a way we can have both.

## My Final Approach (for now)

In the last iteration for now, I've added two more properties to the fake client: An `NSDictionary` named 'response' and an `NSError` named 'error'. In the method `- (RACSignal *)returnSignal` the client now decides, which of its properties it has to return and how.

```objective-c
- (RACSignal *)returnSignal
{
  RACSignal *signal = nil;

  if (self.response) {
    signal = [RACSignal return:self.response];
  } else if (self.error) {
    signal = [RACSignal error:self.error];
  } else {
    signal = self.subject;
  }

  return signal;
}
```

The GET method for instance now returns the result of this method.

```objective-c
- (RACSignal *)GET:(NSString *)urlString parameters:(NSDictionary *)parameters
{
  self.lastUrlString = urlString;
  self.lastParameters = parameters;
  self.getCallCount += 1;

  return [self returnSignal];
}
```

Using this client we are both able to create 'simple' tests which just return one value (either a response or an error) as well as more complicated tests which use one or more `RACSignal` instances that we control separately for each test.

## Closing

I hope I was able to give you some ideas on how to get started with unit testing on ReactiveCocoa-based systems, especially on networking systems. There is - as always - more to the topic than demonstrated here but it should get you started. A thing we didn't cover at all for instance is the check of the properties `lastUrlString` and `lastParameters` which you should do to make sure the call to your service was correct.

Keep on testing, you'll feel better afterwards and sleep better during app launches. And make sure to watch Jon Reid's and Graham Lee's talk once they are online somewhere.
