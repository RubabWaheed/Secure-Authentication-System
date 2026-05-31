# Secure-Authentication-System
```markdown
# 🔐 Secure Authentication System

> A console-based user authentication system in C++ with registration, login, account lockout protection, password hashing, and a separate admin panel.

---

## ✨ Features

| Feature | Description |
|---|---|
| 📝 User Registration | Register with username, hashed password & email validation |
| 🔑 User Login | Secure login with failed attempt tracking |
| 🔒 Account Lockout | Auto-locks after 3 failed login attempts |
| 🔄 Change Password | Logged-in users can update their password |
| 👤 View Profile | See username, email, join date & account status |
| 🛡️ Admin Panel | Admin can view all users & unlock locked accounts |
| 💾 File Persistence | All user data saved to `users.dat` across sessions |

---

## 🛠️ Tech Stack

- **Language:** C++
- **Standard:** C++98 or later
- **Libraries:** `iostream`, `fstream`, `vector`, `string`, `sstream`, `iomanip`
- **Storage:** Flat-file (`users.dat`, pipe-delimited)

---

## 🚀 Getting Started

### Compilation

```bash
g++ -o auth_system auth_system.cpp
```

### Run

```bash
./auth_system
```

---

## 📋 Menu Reference

### 🔓 Logged Out

| Option | Action |
|---|---|
| 1 | Register new user |
| 2 | User login |
| 3 | Admin login |
| 0 | Exit |

### 👤 Logged In (User)

| Option | Action |
|---|---|
| 4 | View profile |
| 5 | Change password |
| 6 | Logout |

### 🛡️ Logged In (Admin)

| Option | Action |
|---|---|
| 4 | View all registered users |
| 5 | Unlock a locked account |
| 6 | Logout |

---

## 🔑 Default Admin Credentials

| Field | Value |
|---|---|
| Password | `admin123` |

> ⚠️ Change `ADMIN_PASS` in source before any real use.

---

## 🔐 Password Rules

- Minimum **6 characters**
- Must contain at least one **uppercase** letter
- Must contain at least one **lowercase** letter
- Must contain at least one **digit**

---

## 🔒 Security Notes

- Passwords are hashed using a **djb2 hash function** (not stored in plaintext)
- Accounts lock after **3 failed login attempts**
- Only admin can unlock locked accounts
- Usernames: 3–20 chars, alphanumeric + `_` and `.` only

> ⚠️ **Disclaimer:** djb2 is not cryptographically secure. For production use, replace with bcrypt or SHA-256.

---

## 💡 How It Works

1. On startup, `loadUsers()` reads `users.dat` and parses all user records
2. Every register/login/update calls `saveUsers()` to persist changes immediately
3. Passwords are never stored — only their hash is saved
4. Failed login counter increments per attempt; resets on successful login

---

## 📁 Data Format (`users.dat`)

```
username|passwordHash|email|failedAttempts|locked|regDate
```

Example:
```
ali123|17289374652|ali@email.com|0|0|2024-5-31
```

---

