---		
layout: post		
title: Modals in Angular 2	Revised
published: false
author: brechtbilliet
comments: true
---

A few months ago I wrote a [blog article](http://blog.brecht.io/Modals-in-angular2/) on how to create modals in Angular 2. The blog post started of with the RC5 version of Angular 2, and later got upgraded to RC6. 

This article is about how I improved/reduced (actually kinda rewrote) the previous code by using **@angular^2.3.0** and the new attachView function of the ApplicationRef class. I was able to remove a lot of code and make it more straightforward. To completely understand the problems this article tries to fix, you might want to check the [previous](http://blog.brecht.io/Modals-in-angular2/) article first.


## The problems I've encountered before
### The AOT problem

To create modals dynamically, we have to create components dynamically.. Makes sense, right?!
Now, Angular works like this: It creates a componentFactory for every component, and uses those factories to actually render the DOM. This is a process that needs to be done, either way. It's how the angular core works... 

There are two possible ways to achieve this. By using JIT(Just in time compilation) which is default, or using [AOT(ahead of time compilation)](https://angular.io/docs/ts/latest/cookbook/aot-compiler.html).
The difference here is that JIT compilation will create these factories when your application bootstraps, and that AOT compilation will use the ngc-compiler as a build step to create them before your application bootstraps. That way, the application will bootstrap faster and the source should be smaller (since the compiler is no longer bundled with the production version of your application).
This is how the application would be bootstrapped:

```typescript
// this is the default way of doing things, (JIT)
import {platformBrowserDynamic} from "@angular/platform-browser-dynamic";
import {DevModule} from "./dev.module"; // this file is writen by a developer
platformBrowserDynamic().bootstrapModule(DevModule);

// this is the AOT way of doings things, (the factories used here are generated by a buildstep)
import {platformBrowser} from "@angular/platform-browser";
import {ProdModuleNgFactory} from "./prod.module.ngfactory"; // this file is generated by ngc
platformBrowser().bootstrapModuleFactory(ProdModuleNgFactory);


```

Let's say that we use AOT and want to create a dynamic modal (which is a dynamic component). In that case we can't access the compiler (because AOT has removed it) and we would need the component factory to create that component dynamically. This means, that for production we had to create our modals differently than for development. That was quite a big frustration since it wasn't pragmatic.

### The viewref problem

As the way we did things before, we needed some kind of reference where we would generate the modal. Therefore we had created a modal-placeholder which registers its viewContainerRef to a modalService that was used to dynamically create modals. So we actually passed a ViewContainerRef to a service, that doesn't sound like a good idea but at the time of writing it was the only way (as far as I knew).

```typescript
export class ModalService {
    private vcRef: ViewContainerRef; // we persist the ref where we should render into
	...
    constructor(private resolver: ComponentFactoryResolver) {
    }

    registerViewContainerRef(vcRef: ViewContainerRef): void {
        this.vcRef = vcRef; // register this ref before we can use it
    }

    create<T>(component: Component<T>){
    	...
    	// resolve the factory
        let componentFactory = this.resolver.resolveComponentFactory<T>(component);
        // generate the component inside the viewref
        let componentRef = this.vcRef.createComponent(componentFactory, ...);
        ...
    }

}
```

That was a lot of hassle, we had to create a modal-placeholder which contained the reference where the modals would be generated. We had to register that reference to a modalservice, check if it would be registered correctly and than create dynamic modals.


## Let's start from scratch
I was at [ng-be](https://ng-be.org/) last week and watched the talk from [Pawel Kozlowski](https://twitter.com/pkozlowski_os) which was about the low level component API's. Watching his talk made me realize creating modals can be much easier (basically the credits of this post should go to him).

So, let's start this from scratch shall we? I would love something like this:

```typescript
let destroyFn = this.modalService.create(MyCustomModalComponent, 
 	// pass in some props
 	{
		firstName: "Brecht",
		lastName: "Billiet"
	});
// we can destroy the modal with the destroyFn if we would want to do that here
```

In this scenario we don't have to register any ref, and we can just generate a modal.
Since Angular 2.3.0, we can actually attach and detach views to ApplicationRef. This allows us to allow components and embedded views to be dirtychecked if they are not placed in any viewContainer.
This means that we don't need to register the ViewContainerRef to the service. We can just append it to the body DOM-tag and attach it to the applicationRef

```typescript
@Injectable()
export class ModalService{
    constructor(
	    private appRef: ApplicationRef // to attach the change detector, 
	    private cfr: ComponentFactoryResolver, // to resolve factories from components 
	    private injector: Injector // to pass the current injector into the modal
	    ) {
    }

    create(component: any, parameters?: Object) {
		// resolve the factory from the component
       const contentCmpFactory = this.cfr.resolveComponentFactory(component);
       // create a componentref and pass the injector so it can use DI
       const componentRef = contentCmpFactory.create(this.injector);
      
       // Append the generated component to the body of our page
       document.querySelector("body").appendChild(
           componentRef.location.nativeElement
       );
       // attach the view to the ApplicationRef, so we have change detection
       this.appRef.attachView(componentRef.hostView);
       
       componentRef.onDestroy(() => {
       		// detach the view on destroy
             this.appRef.detachView(componentRef.hostView);
        });
    }
}
```

### Don't forget entrycomponents

## The final solution
There are a few features most modals require. 
<ul>
<li>Pass @Input's and @Output's to the custom modal</li>
<li>Add component destroy logic</li>
<li>Keep track of the index, (might be needed to stack the modals with z-index etc)</li>
<li>Make it possible to destroy the modal from outside of the modal</li>
</ul>

For me, these are the only features I need and it's not much code.

## Credits
Pawel
