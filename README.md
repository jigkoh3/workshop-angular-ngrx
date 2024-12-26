# การสร้างระบบบริหารจัดการผู้ใช้ด้วย Angular Framework พร้อม NGRX (รวม Authentication) และ UI Styling

## โครงสร้างโครงการ

- ใช้ Angular Framework เป็นโครงสร้างหลัก
- ใช้ NGRX เป็น State Management ทั้ง Authentication และ Users
- ใช้ Angular Material เป็น UI Styling

## 1. สร้างโปรเจกต์ Angular

### 1.1 สร้างโปรเจกต์ Angular ด้วย Angular CLI

```bash
ng new user-management
cd user-management
```

### 1.2 ติดตั้ง Angular Material

```bash
ng add @angular/material
```

### 1.3 ติดตั้ง NGRX

```bash
ng add @ngrx/store
ng add @ngrx/effects
ng add @ngrx/entity
ng add @ngrx/store-devtools
```

### 1.4 ติดตั้ง JWT สำหรับ Authentication

```bash
npm install @auth0/angular-jwt
```

###

## 2. การสร้าง Authentication Module

### 2.1 สร้างโมดูล Authentication

```bash
ng generate module auth --route auth --module app.module
```

### 2.2 สร้าง State Management สำหรับ Authentication

```bash
ng generate feature auth/Auth --module auth/auth.module
```

### 2.3 โค้ด State Management สำหรับ Authentication

#### auth.actions.ts

```bash
import { createAction, props } from '@ngrx/store';

export const login = createAction('[Auth] Login', props<{ email: string; password: string }>());
export const loginSuccess = createAction('[Auth] Login Success', props<{ token: string }>());
export const loginFailure = createAction('[Auth] Login Failure', props<{ error: string }>());
export const logout = createAction('[Auth] Logout');
```

#### auth.reducer.ts

```bash
import { createReducer, on } from '@ngrx/store';
import { login, loginSuccess, loginFailure, logout } from './auth.actions';

export interface AuthState {
  isAuthenticated: boolean;
  token: string | null;
  error: string | null;
}

const initialState: AuthState = {
  isAuthenticated: false,
  token: null,
  error: null,
};

export const authReducer = createReducer(
  initialState,
  on(login, (state) => ({ ...state, error: null })),
  on(loginSuccess, (state, { token }) => ({
    ...state,
    isAuthenticated: true,
    token,
    error: null,
  })),
  on(loginFailure, (state, { error }) => ({
    ...state,
    isAuthenticated: false,
    token: null,
    error,
  })),
  on(logout, () => initialState)
);
```

#### auth.effects.ts

```bash
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { AuthService } from './auth.service';
import { login, loginSuccess, loginFailure, logout } from './auth.actions';
import { catchError, map, mergeMap, tap } from 'rxjs/operators';
import { of } from 'rxjs';
import { Router } from '@angular/router';

@Injectable()
export class AuthEffects {
  login$ = createEffect(() =>
    this.actions$.pipe(
      ofType(login),
      mergeMap(({ email, password }) =>
        this.authService.login(email, password).pipe(
          map((response) => loginSuccess({ token: response.token })),
          catchError((error) => of(loginFailure({ error: error.message })))
        )
      )
    )
  );

  loginSuccess$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(loginSuccess),
        tap(({ token }) => {
          localStorage.setItem('token', token);
          this.router.navigate(['/users']);
        })
      ),
    { dispatch: false }
  );

  logout$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(logout),
        tap(() => {
          localStorage.removeItem('token');
          this.router.navigate(['/auth/login']);
        })
      ),
    { dispatch: false }
  );

  constructor(
    private actions$: Actions,
    private authService: AuthService,
    private router: Router
  ) {}
}
```

### 2.4 สร้าง Auth Service

```bash
ng generate service auth/auth
```

#### auth.service.ts

```bash
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root',
})
export class AuthService {
  private apiUrl = '/api/auth';

  constructor(private http: HttpClient) {}

  login(email: string, password: string): Observable<{ token: string }> {
    return this.http.post<{ token: string }>(`${this.apiUrl}/login`, {
      email,
      password,
    });
  }
}
```

### 2.5 สร้าง Login Component

```bash
ng generate component auth/login
```

#### login.component.ts

```bash
import { Component } from '@angular/core';
import { Store } from '@ngrx/store';
import { login } from '../state/auth.actions';

@Component({
  selector: 'app-login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css'],
})
export class LoginComponent {
  email = '';
  password = '';

  constructor(private store: Store) {}

  onLogin(): void {
    this.store.dispatch(login({ email: this.email, password: this.password }));
  }
}
```

#### login.component.html

```bash
<mat-card>
  <h1>Login</h1>
  <form (ngSubmit)="onLogin()">
    <mat-form-field appearance="fill">
      <mat-label>Email</mat-label>
      <input matInput [(ngModel)]="email" name="email" required />
    </mat-form-field>

    <mat-form-field appearance="fill">
      <mat-label>Password</mat-label>
      <input matInput type="password" [(ngModel)]="password" name="password" required />
    </mat-form-field>

    <button mat-raised-button color="primary" type="submit">Login</button>
  </form>
</mat-card>
```

### 2.6 เพิ่ม Auth Guard เพื่อป้องกัน Route ที่ต้องการ Authentication

```bash
ng generate guard auth/auth
```

#### auth.guard.ts

```bash
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { Store } from '@ngrx/store';
import { selectIsAuthenticated } from './state/auth.selectors';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({
  providedIn: 'root',
})
export class AuthGuard implements CanActivate {
  constructor(private store: Store, private router: Router) {}

  canActivate(): Observable<boolean> {
    return this.store.select(selectIsAuthenticated).pipe(
      map((isAuthenticated) => {
        if (!isAuthenticated) {
          this.router.navigate(['/auth/login']);
          return false;
        }
        return true;
      })
    );
  }
}
```

## 3. การสร้าง Users Module
### 3.1 สร้างโมดูล Users

```bash
ng generate module users --route users --module app.module
```
### 3.2 สร้าง State Management สำหรับ Users

```bash
ng generate feature users/User --module users/users.module
```

#### user.model.ts

```bash
export interface User {
  id: string;
  name: string;
  email: string;
  role: 'Admin' | 'User';
}
```

#### user.actions.ts

```bash
import { createAction, props } from '@ngrx/store';
import { User } from './user.model';

export const loadUsers = createAction('[User] Load Users');
export const loadUsersSuccess = createAction('[User] Load Users Success', props<{ users: User[] }>());
export const loadUsersFailure = createAction('[User] Load Users Failure', props<{ error: string }>());

export const addUser = createAction('[User] Add User', props<{ user: User }>());
export const addUserSuccess = createAction('[User] Add User Success', props<{ user: User }>());
export const addUserFailure = createAction('[User] Add User Failure', props<{ error: string }>());

export const updateUser = createAction('[User] Update User', props<{ user: User }>());
export const updateUserSuccess = createAction('[User] Update User Success', props<{ user: User }>());
export const updateUserFailure = createAction('[User] Update User Failure', props<{ error: string }>());

export const deleteUser = createAction('[User] Delete User', props<{ userId: string }>());
export const deleteUserSuccess = createAction('[User] Delete User Success', props<{ userId: string }>());
export const deleteUserFailure = createAction('[User] Delete User Failure', props<{ error: string }>());
```
#### user.reducer.ts

```bash
import { createReducer, on } from '@ngrx/store';
import {
  loadUsers,
  loadUsersSuccess,
  loadUsersFailure,
  addUser,
  addUserSuccess,
  addUserFailure,
  updateUser,
  updateUserSuccess,
  updateUserFailure,
  deleteUser,
  deleteUserSuccess,
  deleteUserFailure,
} from './user.actions';
import { User } from './user.model';

export interface UserState {
  users: User[];
  loading: boolean;
  error: string | null;
}

const initialState: UserState = {
  users: [],
  loading: false,
  error: null,
};

export const userReducer = createReducer(
  initialState,
  on(loadUsers, (state) => ({ ...state, loading: true, error: null })),
  on(loadUsersSuccess, (state, { users }) => ({ ...state, loading: false, users })),
  on(loadUsersFailure, (state, { error }) => ({ ...state, loading: false, error })),

  on(addUser, (state) => ({ ...state, loading: true })),
  on(addUserSuccess, (state, { user }) => ({
    ...state,
    loading: false,
    users: [...state.users, user],
  })),
  on(addUserFailure, (state, { error }) => ({ ...state, loading: false, error })),

  on(updateUser, (state) => ({ ...state, loading: true })),
  on(updateUserSuccess, (state, { user }) => ({
    ...state,
    loading: false,
    users: state.users.map((u) => (u.id === user.id ? user : u)),
  })),
  on(updateUserFailure, (state, { error }) => ({ ...state, loading: false, error })),

  on(deleteUser, (state) => ({ ...state, loading: true })),
  on(deleteUserSuccess, (state, { userId }) => ({
    ...state,
    loading: false,
    users: state.users.filter((u) => u.id !== userId),
  })),
  on(deleteUserFailure, (state, { error }) => ({ ...state, loading: false, error }))
);
```

#### user.selectors.ts

```bash
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { UserService } from './user.service';
import {
  loadUsers,
  loadUsersSuccess,
  loadUsersFailure,
  addUser,
  addUserSuccess,
  addUserFailure,
  updateUser,
  updateUserSuccess,
  updateUserFailure,
  deleteUser,
  deleteUserSuccess,
  deleteUserFailure,
} from './user.actions';
import { catchError, map, mergeMap } from 'rxjs/operators';
import { of } from 'rxjs';

@Injectable()
export class UserEffects {
  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUsers),
      mergeMap(() =>
        this.userService.getUsers().pipe(
          map((users) => loadUsersSuccess({ users })),
          catchError((error) => of(loadUsersFailure({ error: error.message })))
        )
      )
    )
  );

  addUser$ = createEffect(() =>
    this.actions$.pipe(
      ofType(addUser),
      mergeMap(({ user }) =>
        this.userService.addUser(user).pipe(
          map((newUser) => addUserSuccess({ user: newUser })),
          catchError((error) => of(addUserFailure({ error: error.message })))
        )
      )
    )
  );

  updateUser$ = createEffect(() =>
    this.actions$.pipe(
      ofType(updateUser),
      mergeMap(({ user }) =>
        this.userService.updateUser(user).pipe(
          map((updatedUser) => updateUserSuccess({ user: updatedUser })),
          catchError((error) => of(updateUserFailure({ error: error.message })))
        )
      )
    )
  );

  deleteUser$ = createEffect(() =>
    this.actions$.pipe(
      ofType(deleteUser),
      mergeMap(({ userId }) =>
        this.userService.deleteUser(userId).pipe(
          map(() => deleteUserSuccess({ userId })),
          catchError((error) => of(deleteUserFailure({ error: error.message })))
        )
      )
    )
  );

  constructor(private actions$: Actions, private userService: UserService) {}
}
```

#### user.effects.ts

```bash
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { UserService } from './user.service';
import {
  loadUsers,
  loadUsersSuccess,
  loadUsersFailure,
  addUser,
  addUserSuccess,
  addUserFailure,
  updateUser,
  updateUserSuccess,
  updateUserFailure,
  deleteUser,
  deleteUserSuccess,
  deleteUserFailure,
} from './user.actions';
import { catchError, map, mergeMap } from 'rxjs/operators';
import { of } from 'rxjs';

@Injectable()
export class UserEffects {
  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUsers),
      mergeMap(() =>
        this.userService.getUsers().pipe(
          map((users) => loadUsersSuccess({ users })),
          catchError((error) => of(loadUsersFailure({ error: error.message })))
        )
      )
    )
  );

  addUser$ = createEffect(() =>
    this.actions$.pipe(
      ofType(addUser),
      mergeMap(({ user }) =>
        this.userService.addUser(user).pipe(
          map((newUser) => addUserSuccess({ user: newUser })),
          catchError((error) => of(addUserFailure({ error: error.message })))
        )
      )
    )
  );

  updateUser$ = createEffect(() =>
    this.actions$.pipe(
      ofType(updateUser),
      mergeMap(({ user }) =>
        this.userService.updateUser(user).pipe(
          map((updatedUser) => updateUserSuccess({ user: updatedUser })),
          catchError((error) => of(updateUserFailure({ error: error.message })))
        )
      )
    )
  );

  deleteUser$ = createEffect(() =>
    this.actions$.pipe(
      ofType(deleteUser),
      mergeMap(({ userId }) =>
        this.userService.deleteUser(userId).pipe(
          map(() => deleteUserSuccess({ userId })),
          catchError((error) => of(deleteUserFailure({ error: error.message })))
        )
      )
    )
  );

  constructor(private actions$: Actions, private userService: UserService) {}
}
```

#### user.service.ts

```bash
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { User } from './user.model';

@Injectable({
  providedIn: 'root',
})
export class UserService {
  private apiUrl = '/api/users';

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }

  addUser(user: User): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }

  updateUser(user: User): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/${user.id}`, user);
  }

  deleteUser(userId: string): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${userId}`);
  }
}
```

### 3.3 สร้าง User List Component

```bash
ng generate component users/user-list
```
#### user-list.component.ts

```bash
import { Component, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';
import { loadUsers, deleteUser } from '../state/user.actions';
import { selectAllUsers } from '../state/user.selectors';
import { Observable } from 'rxjs';
import { User } from '../state/user.model';

@Component({
  selector: 'app-user-list',
  templateUrl: './user-list.component.html',
  styleUrls: ['./user-list.component.css'],
})
export class UserListComponent implements OnInit {
  users$: Observable<User[]>;

  constructor(private store: Store) {
    this.users$ = this.store.select(selectAllUsers);
  }

  ngOnInit(): void {
    this.store.dispatch(loadUsers());
  }

  deleteUser(userId: string): void {
    this.store.dispatch(deleteUser({ userId }));
  }
}
```

#### user-list.component.html

```bash
<mat-card>
  <h1>User Management</h1>
  <button mat-raised-button color="primary" routerLink="/users/form">Add User</button>
  <mat-table [dataSource]="users$ | async" class="mat-elevation-z8">
    <ng-container matColumnDef="name">
      <mat-header-cell *matHeaderCellDef> Name </mat-header-cell>
      <mat-cell *matCellDef="let user"> {{ user.name }} </mat-cell>
    </ng-container>

    <ng-container matColumnDef="email">
      <mat-header-cell *matHeaderCellDef> Email </mat-header-cell>
      <mat-cell *matCellDef="let user"> {{ user.email }} </mat-cell>
    </ng-container>

    <ng-container matColumnDef="actions">
      <mat-header-cell *matHeaderCellDef> Actions </mat-header-cell>
      <mat-cell *matCellDef="let user">
        <button mat-icon-button color="warn" (click)="deleteUser(user.id)">
          <mat-icon>delete</mat-icon>
        </button>
        <button mat-icon-button color="accent" routerLink="/users/form/{{ user.id }}">
          <mat-icon>edit</mat-icon>
        </button>
      </mat-cell>
    </ng-container>

    <mat-header-row *matHeaderRowDef="['name', 'email', 'actions']"></mat-header-row>
    <mat-row *matRowDef="let row; columns: ['name', 'email', 'actions']"></mat-row>
  </mat-table>
</mat-card>
```

### 3.4 สร้าง User Form Component

```bash
ng generate component users/user-form
```
#### user-form.component.ts

```bash
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { Store } from '@ngrx/store';
import { addUser, updateUser } from '../state/user.actions';
import { ActivatedRoute, Router } from '@angular/router';
import { User } from '../state/user.model';

@Component({
  selector: 'app-user-form',
  templateUrl: './user-form.component.html',
  styleUrls: ['./user-form.component.css'],
})
export class UserFormComponent implements OnInit {
  userForm: FormGroup;
  editMode = false;
  userId: string | null = null;

  constructor(
    private fb: FormBuilder,
    private store: Store,
    private route: ActivatedRoute,
    private router: Router
  ) {
    this.userForm = this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
      role: ['User', Validators.required],
    });
  }

  ngOnInit(): void {
    this.route.params.subscribe((params) => {
      if (params['id']) {
        this.editMode = true;
        this.userId = params['id'];
        // Load the user data for editing (can be fetched from state or API)
      }
    });
  }

  onSubmit(): void {
    if (this.userForm.valid) {
      const user: User = { ...this.userForm.value, id: this.userId || new Date().getTime().toString() };
      if (this.editMode) {
        this.store.dispatch(updateUser({ user }));
      } else {
        this.store.dispatch(addUser({ user }));
      }
      this.router.navigate(['/users']);
    }
  }
}
```

#### user-form.component.html

```bash 
<mat-card>
  <h1>{{ editMode ? 'Edit User' : 'Add User' }}</h1>
  <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
    <mat-form-field appearance="fill">
      <mat-label>Name</mat-label>
      <input matInput formControlName="name" />
      <mat-error *ngIf="userForm.get('name')?.invalid">Name is required</mat-error>
    </mat-form-field>

    <mat-form-field appearance="fill">
      <mat-label>Email</mat-label>
      <input matInput formControlName="email" />
      <mat-error *ngIf="userForm.get('email')?.invalid">Valid email is required</mat-error>
    </mat-form-field>

    <mat-form-field appearance="fill">
      <mat-label>Role</mat-label>
      <mat-select formControlName="role">
        <mat-option value="User">User</mat-option>
        <mat-option value="Admin">Admin</mat-option>
      </mat-select>
    </mat-form-field>

    <button mat-raised-button color="primary" type="submit">{{ editMode ? 'Update' : 'Save' }}</button>
    <button mat-button color="warn" routerLink="/users">Cancel</button>
  </form>
</mat-card>
```
### เพิ่ม Routing สำหรับ Components ใน Users Module

#### users-routing.module.ts

```bash
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { UserListComponent } from './user-list/user-list.component';
import { UserFormComponent } from './user-form/user-form.component';

const routes: Routes = [
  { path: '', component: UserListComponent },
  { path: 'form', component: UserFormComponent },
  { path: 'form/:id', component: UserFormComponent },
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule],
})
export class UsersRoutingModule {}
```
## 4. การสร้าง Mock API ด้วย JSON Server
```bash
npm install -g json-server
json-server --watch db.json
```
### db.json 
ใน root directory ของโปรเจกต์ (ที่เดียวกับ angular.json)
```bash
{
  "users": [
    {
      "id": "1",
      "name": "John Doe",
      "email": "john.doe@example.com",
      "role": "Admin"
    },
    {
      "id": "2",
      "name": "Jane Smith",
      "email": "jane.smith@example.com",
      "role": "User"
    },
    {
      "id": "3",
      "name": "Alice Johnson",
      "email": "alice.johnson@example.com",
      "role": "User"
    }
  ],
  "auth": {
    "email": "admin@example.com",
    "password": "password123",
    "token": "mock-jwt-token"
  }
}
```
### รัน JSON Server
```bash
json-server --watch db.json --port 3000
```
API จะพร้อมใช้งานที่
* http://localhost:3000/users (สำหรับข้อมูล Users)
* http://localhost:3000/auth (สำหรับข้อมูล Authentication)

### Endpoint Mock API
#### Users API
* GET /users: ดึงรายการผู้ใช้ทั้งหมด
* POST /users: เพิ่มผู้ใช้ใหม่
* PUT /users/:id: แก้ไขข้อมูลผู้ใช้
* DELETE /users/:id: ลบผู้ใช้
### Authentication API
* POST /auth/login: ทำการ Login และส่ง Token กลับมา

### เพิ่ม Middleware สำหรับ Authentication Mock API
หากต้องการจำลองการตรวจสอบ Authentication (ตรวจสอบ Authorization header) คุณสามารถสร้างไฟล์ server.js เพื่อเพิ่ม Middleware และรัน JSON Server แทน
#### server.js
```bash
const jsonServer = require('json-server');
const server = jsonServer.create();
const router = jsonServer.router('db.json');
const middlewares = jsonServer.defaults();

server.use(middlewares);

// Middleware สำหรับ Authentication
server.post('/auth', (req, res) => {
  const { email, password } = req.body;
  if (email === 'admin@example.com' && password === 'password123') {
    res.jsonp({ token: 'mock-jwt-token' });
  } else {
    res.status(401).jsonp({ error: 'Invalid credentials' });
  }
});

// ป้องกันการเข้าถึง /users หากไม่มี Authorization header
server.use((req, res, next) => {
  if (req.path.startsWith('/users') && req.method !== 'GET') {
    const authHeader = req.headers.authorization;
    if (authHeader === 'Bearer mock-jwt-token') {
      next();
    } else {
      res.status(403).jsonp({ error: 'Unauthorized' });
    }
  } else {
    next();
  }
});

// ใช้ Router
server.use(router);

// เริ่ม Server
server.listen(3000, () => {
  console.log('JSON Server is running on http://localhost:3000');
});
```
### รัน Server ด้วย Node.js
```bash
node server.js
```
