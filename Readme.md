# Route Guards
At the moment, any user can navigate anywhere in the application any time, but sometimes you need to control access to different parts of your application for various reasons. Some of which may include the following:
 - Perhaps the user is not authorized to navigate to the target component.
 - Maybe the user must login (authenticate) first.
 - Maybe you should fetch some data before you display the target component.
 - You might want to save pending changes before leaving a component.
 - You might ask the user if it's OK to discard pending changes rather than save them.
You add guards tot the route configuration to handle these scenarios.

A guard's return value controls the routers behavior:
 - If it returns `true`, the navigation process continues.
 - If it returns `false` the current navigation cancels and a new navigation is initiated to the `UrlTree` returned.

The guard might return its boolean answer synchronously. But in many cases, the guard can't produce an answer synchronously. The guard could ask the user a question, save changes to the server, or fetch fresh data. These are all asynchronous operations.

Accordingly, a routing guard can return an `Observable<boolean>` or a `Promise<boolean>` and the router will wait for the observable to resolve to `true` or `false`.

The router supports multiple guard interfaces:
 - `CanActivate` to mediate navigation to a route.
 - `CanActivateChild` to mediate navigation to a child route.
 - `CanDeactivate` to mediate navigation away from the current route.
 - `Resolve` to perform route data retrieval before route activation.
 - `CanLoad` to mediate navigation to a feature module loaded asynchronously.

You can have multiple guards at every level of a routing hierarchy. The router checks the `CanDeactivate` guards first, from the deepest child route to the top. Then it checks the `CanActivate` and `CanActivateChild` guards from the top down to the deepest child route. If the feature module is loaded asynchronously, the CanLoad guard is checked before the module is loaded. If any guard returns false, pending guards that have not completed will be canceled, and the entire navigation is canceled.


## Authentication Service
First we create a Service which will handle the Login/Logout (Authentication) of the user.
```
ng generate service auth
```

<br>
To track the current Authentication State we use an Observable of the type boolean and initialize it with `of(false)`, which means that it is false from default.
```
isLoggedIn: Observable<boolean> = of(false);
```

Then we add a function for the user to login. This code returns an Observable which emits the Value `true`, then pipes it to set the variable isLoggedIn also to true.

`auth.service.ts`
```typescript
login(): Observable<boolean> {
    return of(true).pipe(() => {
      return this.isLoggedIn = of(true)
    });
  }
```

We also need a Function to log the user out. It also navigates the user back to the dashboard

`auth.service.ts`
```typescript
logout(): void {
    this.router.navigate(['/dashboard']);
    this.isLoggedIn = of(false);
  }
```

## Auth Guard
Next we create an AuthGuard. You'll be asked which Interface to use. We're going to use the `canActivate` Option.
```
ng generate guard auth
```
this creates a file that should look like this

`auth.guard.ts`
```TS
import { Injectable } from '@angular/core';
import { ActivatedRouteSnapshot, CanActivate, RouterStateSnapshot, UrlTree } from '@angular/router';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {
  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot): Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree {
    return true;
  }
}
```
Since the Guard implements the `CanActivate` class, it has to implement the `canActivate` method.
This is the method that gets called when a user tries to navigate to a Route that is secured by the Guard. As you can see the method has several return types: `Observable<boolean | UrlTree`, `Observable<Promise<boolean | UrlTree>`, `boolean`, `UrlTree`. If the function returns a boolean/Observable of type Boolean/Promise of Type boolean, and the value is true, the User is allowed to navigate to the requested Page. If the value is false, the User is not allowed to navigate to the requested page and nothing happens. Additionally the method can return an UrlTree, which has the same effect as false, but also navigates the User to the URL, given by the UrlTree.
Now we have to implement our logic for the Guard. We just have to get the Value from our Auth-Service and return it. We can also do additional things like in our example, Logging something to the console and creating an alert.

`app-routing.module.ts`
```ts
canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot): Observable<boolean> {

      let status = false;

      this.authService.isLoggedIn.subscribe(val => {
        val ? console.log("Authorized") : window.alert("Not Authorized");
        return status = val;
      });
      return of(status);
  }
```
Note: We could also just use a simple boolean for the `isLoggedIn` variable, which would drastically reduce the code size in the AuthGuard. However using an Observable makes it asynchronous, which is a good foundation, for when you have a real Authentication Service which will always be asynchronous.

To apply the Guard to a route we have to add it in our Routing Module. First we import it. After that we can just add the `canActivate` attribute to the path and assign our Guard to it.

`app-routing.module.ts`
```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { HeroesComponent } from './heroes/heroes.component';
import { DashboardComponent } from './dashboard/dashboard.component';
import { HeroDetailComponent } from './hero-detail/hero-detail.component';
import { AuthGuard } from './auth.guard';


const routes: Routes = [
  { path: 'heroes', component: HeroesComponent },
  { path: 'detail/:id', component: HeroDetailComponent, canActivate: [AuthGuard] },
  { path: 'dashboard', component: DashboardComponent },
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
]

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```
Note that the `canActivate` attribute takes an Array as the argument. This means you can provide several Guards to protect the Route.


Now if you try to navigate to the hero-detail page, either within the application or by url, you should get an alert saying that your not authorized.
Next thing to do is to let the user sign in.
Create a new Component named login 

`ng generate component login`

Add two methods for logging in and logging out in and in them call the AuthService's methods.
<br>

`login.component.ts`
```ts
login():void {
    this.authService.login();
}

logOut(): void {
    this.authService.logout();
}
```

Now we need two Buttons in the Template for the User. We want to display the Login button only when the user isn't logged in and vice versa, so we use `*ngIf` to do that.
<br>

`login.component.html`
```html
<p>You're <span *ngIf="!(this.authService.isLoggedIn | async)">not</span> logged in</p>
<div *ngIf="this.authService.isLoggedIn | async">
    <a (click)="logOut()">Logout</a>
</div>
<div *ngIf="!(this.authService.isLoggedIn | async)">
    <a (click)="login()">Login</a>
</div>
```
Note: we used the async pipe in the conditions. That way we don't have to subscribe to the `isLoggedIn` Observable, and the values get automatically updated.