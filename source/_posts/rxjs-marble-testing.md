---
title: rxjs-marble-testing
date: 2020-04-12 20:08:45
tags:
---

## A very typical usage case of Marble Testing for RxJS

### What's Marble Testing

RxJS is a powerful tool to handle various asynchronous tasks. Sometime RxJS observables are hard to test. In the community, there is a kind of particle unit test solution for RxJS observables called marble testing.

For marble testing, you can refer to [these articles](https://medium.com/@bencabanes/marble-testing-observable-introduction-1f5ad39231c). They all do an excellent job of introducing what it is. 


### A real case to show Marble Testing's power

Recently in my development work, I came across a case where I used marble testing properly to solve the issue. 

The background is in the root `index.html` file, there is one third-party javascript library/SDK. For some business reasons, after the third-party library is loaded, we bind some value to the global `window` object. 

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






