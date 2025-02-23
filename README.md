# UserManagement-MEPN

Creating an Angular application with an Express backend, PostgreSQL database, and user management system including security, login, authentication, and session management is a multi-step process. Below is a high-level guide to help you set up the application.

---

### **1. Prerequisites**
Before starting, ensure you have the following installed:
- Node.js and npm
- Angular CLI (`npm install -g @angular/cli`)
- PostgreSQL
- Postman (for testing APIs)

---

### **2. Set Up the Backend (Express + PostgreSQL)**

#### **Step 1: Initialize the Express Project**
1. Create a new directory for your project:
   ```bash
   mkdir angular-express-postgres-app
   cd angular-express-postgres-app
   ```
2. Initialize a new Node.js project:
   ```bash
   npm init -y
   ```
3. Install required dependencies:
   ```bash
   npm install express pg cors body-parser express-session bcrypt dotenv
   ```

#### **Step 2: Set Up PostgreSQL**
1. Create a PostgreSQL database:
   ```sql
   CREATE DATABASE user_manager;
   ```
2. Create a `users` table:
   ```sql
   CREATE TABLE users (
       id SERIAL PRIMARY KEY,
       username VARCHAR(50) UNIQUE NOT NULL,
       password VARCHAR(255) NOT NULL,
       email VARCHAR(100) UNIQUE NOT NULL,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```

#### **Step 3: Create the Express Server**
1. Create a `server.js` file:
   ```javascript
   const express = require('express');
   const bodyParser = require('body-parser');
   const cors = require('cors');
   const session = require('express-session');
   const { Pool } = require('pg');
   const bcrypt = require('bcrypt');
   const dotenv = require('dotenv');

   dotenv.config();

   const app = express();
   const port = 5000;

   // Middleware
   app.use(cors({ origin: 'http://localhost:4200', credentials: true }));
   app.use(bodyParser.json());
   app.use(
       session({
           secret: process.env.SESSION_SECRET,
           resave: false,
           saveUninitialized: true,
           cookie: { secure: false, maxAge: 1000 * 60 * 60 * 24 }, // 1 day
       })
   );

   // PostgreSQL connection
   const pool = new Pool({
       user: process.env.DB_USER,
       host: process.env.DB_HOST,
       database: process.env.DB_NAME,
       password: process.env.DB_PASSWORD,
       port: process.env.DB_PORT,
   });

   // Routes
   app.post('/register', async (req, res) => {
       const { username, email, password } = req.body;
       const hashedPassword = await bcrypt.hash(password, 10);

       try {
           const result = await pool.query(
               'INSERT INTO users (username, email, password) VALUES ($1, $2, $3) RETURNING *',
               [username, email, hashedPassword]
           );
           res.status(201).json(result.rows[0]);
       } catch (err) {
           res.status(500).json({ error: err.message });
       }
   });

   app.post('/login', async (req, res) => {
       const { username, password } = req.body;

       try {
           const result = await pool.query('SELECT * FROM users WHERE username = $1', [username]);
           if (result.rows.length === 0) {
               return res.status(401).json({ error: 'Invalid username or password' });
           }

           const user = result.rows[0];
           const isValidPassword = await bcrypt.compare(password, user.password);

           if (!isValidPassword) {
               return res.status(401).json({ error: 'Invalid username or password' });
           }

           req.session.userId = user.id;
           res.status(200).json({ message: 'Login successful', user });
       } catch (err) {
           res.status(500).json({ error: err.message });
       }
   });

   app.post('/logout', (req, res) => {
       req.session.destroy((err) => {
           if (err) {
               return res.status(500).json({ error: 'Could not log out' });
           }
           res.clearCookie('connect.sid');
           res.status(200).json({ message: 'Logout successful' });
       });
   });

   app.get('/profile', async (req, res) => {
       if (!req.session.userId) {
           return res.status(401).json({ error: 'Unauthorized' });
       }

       try {
           const result = await pool.query('SELECT * FROM users WHERE id = $1', [req.session.userId]);
           res.status(200).json(result.rows[0]);
       } catch (err) {
           res.status(500).json({ error: err.message });
       }
   });

   // Start server
   app.listen(port, () => {
       console.log(`Server running on http://localhost:${port}`);
   });
   ```

3. Create a `.env` file for environment variables:
   ```
   DB_USER=your_db_user
   DB_HOST=localhost
   DB_NAME=user_manager
   DB_PASSWORD=your_db_password
   DB_PORT=5432
   SESSION_SECRET=your_session_secret
   ```

---

### **3. Set Up the Frontend (Angular)**

#### **Step 1: Create an Angular App**
1. Generate a new Angular app:
   ```bash
   ng new angular-frontend
   cd angular-frontend
   ```
2. Install Angular HTTP client:
   ```bash
   npm install @angular/common@latest @angular/core@latest
   ```

#### **Step 2: Create Services and Components**
1. Generate a service for API calls:
   ```bash
   ng generate service auth
   ```
2. Update `auth.service.ts`:
   ```typescript
   import { Injectable } from '@angular/core';
   import { HttpClient } from '@angular/common/http';
   import { Observable } from 'rxjs';

   @Injectable({
       providedIn: 'root',
   })
   export class AuthService {
       private apiUrl = 'http://localhost:5000';

       constructor(private http: HttpClient) {}

       register(user: any): Observable<any> {
           return this.http.post(`${this.apiUrl}/register`, user);
       }

       login(credentials: any): Observable<any> {
           return this.http.post(`${this.apiUrl}/login`, credentials);
       }

       logout(): Observable<any> {
           return this.http.post(`${this.apiUrl}/logout`, {});
       }

       getProfile(): Observable<any> {
           return this.http.get(`${this.apiUrl}/profile`);
       }
   }
   ```

3. Generate components for login, register, and profile:
   ```bash
   ng generate component login
   ng generate component register
   ng generate component profile
   ```

4. Update `app.module.ts` to include routing and HTTP client:
   ```typescript
   import { NgModule } from '@angular/core';
   import { BrowserModule } from '@angular/platform-browser';
   import { HttpClientModule } from '@angular/common/http';
   import { FormsModule } from '@angular/forms';
   import { AppRoutingModule } from './app-routing.module';
   import { AppComponent } from './app.component';
   import { LoginComponent } from './login/login.component';
   import { RegisterComponent } from './register/register.component';
   import { ProfileComponent } from './profile/profile.component';

   @NgModule({
       declarations: [AppComponent, LoginComponent, RegisterComponent, ProfileComponent],
       imports: [BrowserModule, HttpClientModule, FormsModule, AppRoutingModule],
       providers: [],
       bootstrap: [AppComponent],
   })
   export class AppModule {}
   ```

5. Update `app-routing.module.ts`:
   ```typescript
   import { NgModule } from '@angular/core';
   import { RouterModule, Routes } from '@angular/router';
   import { LoginComponent } from './login/login.component';
   import { RegisterComponent } from './register/register.component';
   import { ProfileComponent } from './profile/profile.component';

   const routes: Routes = [
       { path: 'login', component: LoginComponent },
       { path: 'register', component: RegisterComponent },
       { path: 'profile', component: ProfileComponent },
       { path: '', redirectTo: '/login', pathMatch: 'full' },
   ];

   @NgModule({
       imports: [RouterModule.forRoot(routes)],
       exports: [RouterModule],
   })
   export class AppRoutingModule {}
   ```

6. Update `login.component.ts`:
   ```typescript
   import { Component } from '@angular/core';
   import { AuthService } from '../auth.service';
   import { Router } from '@angular/router';

   @Component({
       selector: 'app-login',
       templateUrl: './login.component.html',
       styleUrls: ['./login.component.css'],
   })
   export class LoginComponent {
       credentials = { username: '', password: '' };

       constructor(private authService: AuthService, private router: Router) {}

       login() {
           this.authService.login(this.credentials).subscribe(
               (response) => {
                   this.router.navigate(['/profile']);
               },
               (error) => {
                   console.error('Login failed', error);
               }
           );
       }
   }
   ```

7. Update `login.component.html`:
   ```html
   <h2>Login</h2>
   <form (ngSubmit)="login()">
       <input [(ngModel)]="credentials.username" name="username" placeholder="Username" required />
       <input [(ngModel)]="credentials.password" name="password" type="password" placeholder="Password" required />
       <button type="submit">Login</button>
   </form>
   ```

---

### **4. Run the Application**
1. Start the Express server:
   ```bash
   node server.js
   ```
2. Start the Angular app:
   ```bash
   ng serve
   ```
3. Access the app at `http://localhost:4200`.

---

This is a basic implementation. You can expand it by adding features like password reset, email verification, and role-based access control.
