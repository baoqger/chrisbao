---
title:  Typical usage case of Marble Testing for RxJS
date: 2020-04-12 20:08:45
tags:
---

### What's Marble Testing

RxJS is a powerful tool to handle various asynchronous tasks. Sometime RxJS observables are hard to test. In the community, there is one particular unit testing solution for RxJS observables called marble testing.

For marble testing, you can refer to [these articles](https://medium.com/@bencabanes/marble-testing-observable-introduction-1f5ad39231c). They all do an excellent job of introducing what it is. 


### A real case to show Marble Testing's power

Recently in my development work, I came across a case where I used marble testing properly to solve the issue. 

The background is for my application in the root `index.html` file, there is one third-party javascript library/SDK. For some business reasons, after the third-party library is loaded, we bind some value to the global `window` object. 

``` javascript
      window.onSecretLibraryLoad = (res) => {
        window.SecretLibrary = res; 
      };
```

Before the loaded event is triggered, the `window.SecretLibary` was `undefiend`. After the event is triggered, it will be assigned some value. 

This global value is vital for my application since one of the UI components depends on it to determine to show or hide. 

Since the script was loaded with `async` flag for performance consideration, this component needs to check the `window.SecretLibrary` value repeatedly. 

``` javascript
export class UIComponent implements OnInit {
    public isSecretLibraryReady$: Observable<boolean>;
    public ngOnInit(): void {
        this.isSecretLibraryReady$ = this.getSecretLibraryLoadStatus();
    }

    public getSecretLibraryLoadStatus(): Observable<boolean> {
        return timer(0, 500).pipe(
            map((t) => ({
                currentTimeTick: t,
                isReady: Boolean((window as any).SecretLibrary)
            })),
            takeWhile(
                (val) => !val.isReady && val.currentTimeTick <= 30000/500,
                true
            ),
            pluck("isReady")
        );
    }
}
```

The interesting part comes from the `getSecretLibraryLoadStatus` function. It utilizes the `timer`, `map`, `takeWhile`, and `pluck` operators of RxJS library to realize the repeatedly status checking and updating purpose. 

RxJS operator is not this article's topic, so I will not dig deep into it. Just add one comment for the `takeWhile` part, which seems a little hard to understand at the first look. It defines two conditions to stop the observable: first case is the `window.SecretLibrary` value is not undefined, which means the script is loaded; the second case is it reaches the maximum time limit (30 seconds). 


### How to test it: marble testing

Traditionally we use `subscribe and assert` pattern to test RxJS observables. But for the case, as mentioned above: time-based observables, the traditional solution can't handle it well. We can't wait 30 seconds when we run the unit tests during local development or in CI pipeline. That's unacceptable.

Marble testing can fix this issue correctly by the concept of virtual time. For what is virtual time, you can refer to [these great articles](https://medium.com/angular-in-depth/how-to-test-observables-a00038c7faad). I will not discuss it at a detailed level in this post. Simply speaking, virtual time empowers us to test asynchronous RxJS observable synchronously. In virtual time asynchronous functions like `setTimeout` and `setInterval` will use fake time instead of actual time. And the magic comes from the `TestScheduler` of RxJS. 

The marble testing solution as following: 

``` javascript
describe("UIComponent", () => {
  let component: UIComponent;
  let fixture: ComponentFixture<UIComponent>;
  let scheduler: TestScheduler;

  beforeEach(async(() => {
    TestBed.configureTestingModule({
      declarations: [UIComponent],
    }).compileComponents();
  }));

  beforeEach(() => {
    fixture = TestBed.createComponent(UIComponent);
    component = fixture.componentInstance;
    scheduler = new TestScheduler((actual, expected) => expect(actual).toEqual(expected));
    (window as any).SecretLibrary = undefined;
  });

  it("getSecretLibraryLoadStatus method should return a boolean observable stream, for script load success case", () => {
    scheduler.run(({ expectObservable }) => {
      timer(2 * 1000).subscribe(() => {
        (window as any).SecretLibrary = "SecretLibrary";
      });
      // tricky point about marble test: a, b, c these placeholders account 1 frame of virtual time
      expectObservable(component.getSecretLibraryLoadStatus()).toBe(
        "a 499ms b 499ms c 499ms d 499ms (e|)",
        {
          a: false,
          b: false,
          c: false,
          d: false,
          e: true
        }
      );
    });
  });
});

```

Let me emphasize several key points to understand the above testing case. `getSecretLibraryLoadStatus` will repeatedly check the status of `window.SecretLibrary` every 500 milliseconds and return a boolean value stream. 

I manually set the `window.SecretLibrary` value after 2 seconds by calling this: 

``` javascript
      timer(2 * 1000).subscribe(() => {
        (window as any).SecretLibrary = "SecretLibrary";
      });
```

So Let's imagine what the result should be: the first four emit value will be `false`,  and the last value is `true`. That's the expected observable. 

But please notice that `scheduler.run`, everything run inside the callback will use virtual time. So we didn't wait 2 seconds for the `timer` operator. Instead, it runs synchronously. 

The second point needs to notice is the syntax of the marble string `a 499ms b 499ms c 499ms d 499ms (e|)`. Yes, it looks a little wired, but be careful that's the correct way to represent marble. **The alphanumeric values represent an actual emitted value, which advances time one virtual frame**.

Here I only show the case where the third-party script load successfully. Of course, to have full testing coverage, we need to consider the negative case where the script failed to load. You can figure it out. 

### Summary

In this post, I didn't go into the detailed syntax of marble testing. I think the reader can find the related document easily. I record a real usage case in the development where marble testing can show its power to handle async observables. 








