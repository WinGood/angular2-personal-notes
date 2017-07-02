# Angular 2 personal notes

## Change detection

Every component gets a change detector responsible for checking the bindings defined in its template. Examples of bindings: {{todo.text}} and [todo]=”t”. Change detectors propagate bindings from the root to leaves in the depth first order.

Angular 2 does not have a generic mechanism implementing two-way data-bindings. That is why the change detection graph is a directed tree and cannot have cycles. This makes the system significantly more performant. And what is more important we gain guarantees that make the system more predictable and easier debug.

### What causes change?

Basically application state change can be caused by three things:

1. Events - click, submit, …
2. XHR - Fetching data from a remote server
3. Timers - setTimeout(), setInterval()

They are all asynchronous. **NgZone** notifies Angular about changes, each component has its own change detector since this allows us to control, for each component individually, how and when change detection is performed!

In Angular’s source code, there’s this thing called ApplicationRef, which listens to **NgZones** onTurnDone event. Whenever this event is fired, it executes a tick() function which essentially performs change detection.

```
// very simplified version of actual source
class ApplicationRef {

  changeDetectorRefs:ChangeDetectorRef[] = [];

  constructor(private zone: NgZone) {
    this.zone.onTurnDone
      .subscribe(() => this.zone.run(() => this.tick());
  }

  tick() {
    this.changeDetectorRefs
      .forEach((ref) => ref.detectChanges());
  }
}
```

### Performance

By default, even if we have to check every single component every single time an event happens, Angular is very fast. It can perform hundreds of thousands of checks within a couple of milliseconds. 

**Angular creates change detector classes at runtime for each component**, which are monomorphic, because they know exactly what the shape of the component’s model is. VMs can perfectly optimize this code, which makes it very fast to execute. 

But wouldn’t it be great if we could tell Angular to only run change detection for the **parts of the application that changed their state?**

It turns out there are data structures that give us some guarantees of when something has changed or not - Immutables and Observables. If we happen to use these structures or types, and we tell Angular about it, change detection can be much much faster.

**Reducing the number of checks**

Angular can skip entire change detection subtrees when input properties don’t change. We just learned that a “change” means “new reference”.  We can tell Angular to skip change detection for this component’s subtree if none of its inputs changed by setting the change detection strategy to **OnPush** like this:

```
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
class VCardCmp {
  @Input() vData;
}
```

**Observables**

As mentioned earlier, Observables also give us some certain guarantees of when a change has happened. Unlike immutable objects, they don’t give us new references when a change is made. Instead, they fire events we can subscribe to in order to react to them.

So, if we use Observables and we want to use OnPush to skip change detector subtrees, but the reference of these objects will never change, how do we deal with that? It turns out Angular has a very smart way to enable paths in the component tree to be checked for certain events, which is exactly what we need in that case.

```
@Component({
  template: '{{counter}}',
  changeDetection: ChangeDetectionStrategy.OnPush
})
class CartBadgeCmp {

  @Input() addItemStream:Observable<any>;
  counter = 0;
  
  constructor(private cd: ChangeDetectorRef) {}

  ngOnInit() {
    this.addItemStream.subscribe(() => {
      this.counter++; // application state changed
      this.cd.markForCheck();
    })
  }
}
```

We can access a component’s ChangeDetectorRef via dependency injection, which comes with an API called markForCheck(). It marks the path from our component until root to be checked for the next change detection run.

**How to turn off change detection for component?**

```
class CartBadgeCmp {
  constructor(private cd: ChangeDetectorRef) {
    ref.detach(); // turn off
    setInterval(() => {
      this.ref.detectChanges();
    }, 5000);
  }
}
```

**Links:**

1. [ANGULAR CHANGE DETECTION EXPLAINED](https://blog.thoughtram.io/angular/2016/02/22/angular-2-change-detection-explained.html)
2. [How does Angular Change Detection Really Work ?](http://blog.angular-university.io/how-does-angular-2-change-detection-really-work/)
3. [Change Detection in Angular](https://vsavkin.com/change-detection-in-angular-2-4f216b855d4c)
4. [Performance Optimization in Angular 2.0 article](https://eyalvardi.wordpress.com/2016/12/20/performance-optimization-in-angular-2-0/)
5. [Performance Optimization in Angular 2.0 demo](http://ng-course.org/ng-course/demos/change_detection/index.html#/home)

## RxJS

1. [Angular: Don't forget to unsubscribe()](http://brianflove.com/2016/12/11/anguar-2-unsubscribe-observables/)
