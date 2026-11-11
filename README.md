# MERN Auth Lab: Authentication + Authorization (simple & readable)

A small MERN project made for **learning**. It shows, in as little code as possible:

| Feature | Where to read it |
|---|---|
| OTP email verification (Nodemailer + Gmail SMTP) | `backend/src/controllers/authController.js`, `utils/otp.js`, `utils/sendEmail.js`, `config/mailer.js` |
| JWT in an HTTP-only cookie | `backend/src/utils/token.js` |
| Server-side session management | `backend/src/models/Session.js`, `middleware/authMiddleware.js` |
| Role-based authorization (user / moderator / admin) | `backend/src/middleware/authMiddleware.js` (`authorize`), `routes/userRoutes.js` |
| Frontend MVC + route guards | `frontend/src/models`, `controllers`, `views`, `routes` |

---

## 1. Folder structure (MVC on both sides)

```
mern-auth/
├── backend/
│   ├── server.js                  # app entry: middleware + routes
│   ├── scripts/seedAdmin.js       # creates the first admin
│   └── src/
│       ├── config/                # db.js, mailer.js (Gmail SMTP)
│       ├── models/                # M: User.js, Session.js
│       ├── controllers/           # C: authController.js, userController.js
│       ├── routes/                # URL -> controller (+ protect/authorize)
│       ├── middleware/            # protect, authorize, error handlers
│       └── utils/                 # otp, token/cookie, sendEmail helpers
└── frontend/
    └── src/
        ├── models/                # M: API calls (axios) - talks to backend
        ├── controllers/           # C: AuthController.jsx (state + actions)
        ├── views/                 # V: components/ and pages/
        ├── routes/                # AppRoutes.jsx (+ guards)
        └── styles/index.css
```

*(The backend "View" is the JSON response; the frontend "View" is the React pages.)*

---

## 2. Install and run

### Requirements
- Node.js 18+
- MongoDB running locally (`mongodb://127.0.0.1:27017`) **or** a free MongoDB Atlas connection string
- A Gmail account with an **App Password** (see step 2.1)

### 2.1 Create a Gmail App Password (needed for Nodemailer)
1. Open your Google Account -> **Security**.
2. Turn **2-Step Verification ON** (required).
3. Search for **App passwords** (https://myaccount.google.com/apppasswords).
4. Create one (name it "MERN Auth") -> Google shows a **16-character password**.
5. Copy it (no spaces) -> this goes in `EMAIL_PASS`.

### 2.2 Backend
```bash
cd backend
npm install
cp .env.example .env        # Windows: copy .env.example .env
```
Open `backend/.env` and fill in: `MONGO_URI`, `JWT_SECRET`, `EMAIL_USER`, `EMAIL_PASS`.

Create the first admin (optional but recommended), then start:
```bash
npm run seed:admin          # admin@example.com / Admin@12345 (change in .env)
npm run dev                 # runs on http://localhost:5000
```

### 2.3 Frontend (new terminal)
```bash
cd frontend
npm install
npm run dev                 # opens on http://localhost:5173
```

Open **http://localhost:5173**.

---

## 3. Try it (suggested order)

1. **Register** with a real email you can open -> you get a 6-digit code.
2. Enter the code -> you're verified **and** logged in -> Dashboard.
3. Dashboard shows your role (`user`) and which pages it can open. Visit `/admin` -> redirected to *403*.
4. Log out, log in with the **seeded admin** -> open **Admin panel** -> promote your first user to `moderator`.
5. Log in as that user -> **Moderation** page now opens; `/admin` is still blocked.
6. Log in from two browsers -> on one, open Dashboard and **Revoke** the other session -> the other browser is logged out on its next request.
7. Open DevTools -> Application -> Cookies: the `token` cookie is marked **HttpOnly**. In the console, `document.cookie` will not show it.

---

## 4. The flows, step by step

### A) Registration + OTP
```
Browser                      Express                          MongoDB / Gmail
  | POST /auth/register ->     |                                   |
  |                            | create User {isVerified:false}    |
  |                            | otp = random 6 digits             |
  |                            | store HMAC(otp) + expiry   ------>|
  |                            | Nodemailer sends otp by email ----> (Gmail SMTP)
  | <- 201 {email}             |                                   |
  | POST /auth/verify-otp ->   |                                   |
  |                            | compare HMAC(input) to stored hash|
  |                            | (max 5 wrong tries, 10 min expiry)|
  |                            | isVerified = true                 |
  |                            | create Session + set cookie       |
  | <- 200 {user} + Set-Cookie |                                   |
```
Key ideas: the OTP is **hashed** in the DB; it **expires**; wrong guesses are **limited**; resend has a **60s cooldown**; and the client can never choose its own `role` (the server ignores it).

### B) Login = JWT + server-side session
```
POST /auth/login {email,password}
  1. find user, bcrypt.compare(password)
  2. not verified? -> send new OTP, reply 403 {needsVerification}
  3. Session.create({user, userAgent, ip, expiresAt})   <- the server-side session
  4. token = JWT({ id: userId, sid: session._id })       <- JWT only points to the session
  5. res.cookie('token', token, { httpOnly, sameSite, secure(prod) })
```

### C) Every protected request (`protect` middleware)
```
Cookie -> JWT valid? -> Session row still in DB? -> User still exists? -> req.user set -> next()
   no -> 401        no -> 401 (logged out / revoked)        no -> 401
```
**Why both a JWT and a session?** A JWT alone cannot be cancelled before it expires. Because each JWT points to a session row we control, deleting that row (logout, revoke, admin deletes user) kills access **immediately**.

### D) Authorization (`authorize` middleware)
```js
router.get('/', protect, authorize('admin'), getUsers);
//              ^ who are you?   ^ are you allowed?
```
Role is read from the **database** on every request (not stored in the JWT), so changing someone's role applies instantly.

### E) Frontend side
- The React app **never sees the JWT**; the browser stores and sends the cookie automatically (`withCredentials: true`).
- After refresh, React state is empty, so `AuthController` calls `GET /auth/me`. If the cookie + session are valid, the user is restored.
- `ProtectedRoute` / `GuestRoute` redirect for good UX. **They are not security**: the real checks are on the server.

---

## 5. Reusing this in your own project (checklist)

**Backend**
1. Copy `models/User.js`, `models/Session.js`, `utils/*`, `config/mailer.js`, `middleware/*`.
2. Copy `controllers/authController.js` + `routes/authRoutes.js`; mount at `/api/auth`.
3. Add `cookieParser()`, `express.json()`, and `cors({origin, credentials:true})` in `server.js`.
4. Protect any route with `protect`, restrict with `authorize('role')`.

**Frontend**
1. Copy `models/api.js` + `models/authModel.js`, `controllers/AuthController.jsx`.
2. Wrap your app in `<AuthProvider>`.
3. Use `ProtectedRoute` / `GuestRoute` in your routes.
4. Axios must use `withCredentials: true`.

---

## 6. Production notes (things deliberately kept out to stay simple)
- Serve over **HTTPS** and set `NODE_ENV=production` (cookie becomes `secure`).
- If frontend and backend are on different domains, use `sameSite: 'none'` + `secure: true` and keep CORS `credentials: true`.
- Add rate limiting (`express-rate-limit`) on `/login` and `/register`.
- Add CSRF protection if you rely on cross-site cookies; `sameSite: 'lax'` covers common cases.
- Add "forgot password" (same OTP idea, different purpose).

## 7. Troubleshooting
| Problem | Fix |
|---|---|
| "Could not send the OTP email" | Wrong `EMAIL_USER`/`EMAIL_PASS`. Use a Gmail **App Password**, 2-Step Verification must be on |
| `MongoDB connection failed` | Start MongoDB, or paste your Atlas URI in `MONGO_URI` |
| Always logged out after login | Open the app via `http://localhost:5173` (not 127.0.0.1) and keep the Vite proxy |
| Port in use | Change `PORT` in `.env` and the proxy target in `frontend/vite.config.js` |
