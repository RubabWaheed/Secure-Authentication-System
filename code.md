```cpp
/*
 * Secure Authentication System
 * User registration/login + Admin login with password
 */

#include <iostream>
#include <iomanip>
#include <fstream>
#include <vector>
#include <string>
#include <sstream>
#include <ctime>
#include <cstdlib>

using namespace std;

const string DB_FILE = "users.dat";
const string DELIM = "|";
const string ADMIN_PASS = "admin123";
const int MAX_ATTEMPTS = 3;

struct User {
    string username;
    string passwordHash;
    string email;
    string regDate;
    int failedAttempts;
    bool locked;
};

string currentUser = "";
bool isLoggedIn = false;
bool isAdmin = false;

string intToStr(int n) {
    stringstream ss;
    ss << n;
    return ss.str();
}

string getDate() {
    time_t n = time(NULL);
    tm* l = localtime(&n);
    stringstream ss;
    ss << 1900+l->tm_year << "-" << 1+l->tm_mon << "-" << l->tm_mday;
    return ss.str();
}

string hashPass(const string& pw) {
    unsigned long h = 5381;
    for(size_t i = 0; i < pw.length(); i++) {
        h = ((h << 5) + h) + (unsigned char)pw[i];
    }
    stringstream ss;
    ss << h;
    return ss.str();
}

vector<User> loadUsers() {
    vector<User> list;
    ifstream in(DB_FILE.c_str());
    if(!in) return list;

    string line;
    while(getline(in, line)) {
        if(line.empty()) continue;
        User u;
        size_t p1 = line.find(DELIM);
        size_t p2 = line.find(DELIM, p1 + DELIM.length());
        size_t p3 = line.find(DELIM, p2 + DELIM.length());
        size_t p4 = line.find(DELIM, p3 + DELIM.length());
        size_t p5 = line.find(DELIM, p4 + DELIM.length());

        if(p1 == string::npos || p2 == string::npos || p3 == string::npos) continue;

        u.username = line.substr(0, p1);
        u.passwordHash = line.substr(p1 + DELIM.length(), p2 - p1 - DELIM.length());
        u.email = line.substr(p2 + DELIM.length(), p3 - p2 - DELIM.length());

        string fa = (p4 != string::npos) ? line.substr(p3 + DELIM.length(), p4 - p3 - DELIM.length()) : "0";
        u.failedAttempts = atoi(fa.c_str());

        string lk = (p5 != string::npos) ? line.substr(p4 + DELIM.length(), p5 - p4 - DELIM.length()) : "0";
        u.locked = (lk == "1");

        u.regDate = (p5 != string::npos) ? line.substr(p5 + DELIM.length()) : getDate();

        list.push_back(u);
    }
    in.close();
    return list;
}

void saveUsers(const vector<User>& list) {
    ofstream out(DB_FILE.c_str());
    if(!out) {
        cout << "ERROR: Cannot write to database." << endl;
        return;
    }
    for(size_t i = 0; i < list.size(); i++) {
        out << list[i].username << DELIM
            << list[i].passwordHash << DELIM
            << list[i].email << DELIM
            << list[i].failedAttempts << DELIM
            << (list[i].locked ? "1" : "0") << DELIM
            << list[i].regDate << endl;
    }
    out.close();
}

bool userExists(const string& name, const vector<User>& list) {
    for(size_t i = 0; i < list.size(); i++) {
        if(list[i].username == name) return true;
    }
    return false;
}

bool validUsername(const string& name) {
    if(name.length() < 3 || name.length() > 20) return false;
    for(size_t i = 0; i < name.length(); i++) {
        char c = name[i];
        if(!isalnum((unsigned char)c) && c != '_' && c != '.') return false;
    }
    return true;
}

bool validPassword(const string& pw) {
    if(pw.length() < 6) return false;
    bool up = false, low = false, dig = false;
    for(size_t i = 0; i < pw.length(); i++) {
        if(isupper((unsigned char)pw[i])) up = true;
        else if(islower((unsigned char)pw[i])) low = true;
        else if(isdigit((unsigned char)pw[i])) dig = true;
    }
    return up && low && dig;
}

bool validEmail(const string& em) {
    size_t at = em.find('@');
    if(at == string::npos || at == 0) return false;
    size_t dot = em.find('.', at);
    return (dot != string::npos && dot > at + 1 && dot < em.length() - 1);
}

void registerUser() {
    cout << "\n========================================" << endl;
    cout << "      NEW USER REGISTRATION" << endl;
    cout << "========================================" << endl;

    vector<User> users = loadUsers();
    User u;

    cout << "\nUsername (3-20 chars): ";
    cin >> u.username;
    while(!validUsername(u.username)) {
        cout << "Invalid format. Try again: ";
        cin >> u.username;
    }
    if(userExists(u.username, users)) {
        cout << "\nERROR: Username already taken.\n" << endl;
        return;
    }

    string pw, confirm;
    cout << "Password (6+ chars, upper+lower+digit): ";
    cin >> pw;
    while(!validPassword(pw)) {
        cout << "Weak password. Must contain uppercase, lowercase, digit.\n";
        cout << "Password: ";
        cin >> pw;
    }
    cout << "Confirm password: ";
    cin >> confirm;
    while(confirm != pw) {
        cout << "Mismatch. Confirm: ";
        cin >> confirm;
    }

    cout << "Email: ";
    cin >> u.email;
    while(!validEmail(u.email)) {
        cout << "Invalid email. Try again: ";
        cin >> u.email;
    }

    u.passwordHash = hashPass(pw);
    u.regDate = getDate();
    u.failedAttempts = 0;
    u.locked = false;

    users.push_back(u);
    saveUsers(users);
    cout << "\nAccount created successfully!\n" << endl;
}

void userLogin() {
    cout << "\n========================================" << endl;
    cout << "           USER LOGIN" << endl;
    cout << "========================================" << endl;

    if(isLoggedIn) {
        cout << "Already logged in as " << currentUser << ".\n" << endl;
        return;
    }

    string name, pw;
    cout << "\nUsername: ";
    cin >> name;

    vector<User> users = loadUsers();
    int idx = -1;
    for(size_t i = 0; i < users.size(); i++) {
        if(users[i].username == name) {
            idx = (int)i;
            break;
        }
    }

    if(idx == -1) {
        cout << "\nERROR: User not found.\n" << endl;
        return;
    }

    if(users[idx].locked) {
        cout << "\nACCOUNT LOCKED. Contact admin.\n" << endl;
        return;
    }

    cout << "Password: ";
    cin >> pw;

    if(hashPass(pw) != users[idx].passwordHash) {
        users[idx].failedAttempts++;
        cout << "\nInvalid password. Attempt " << users[idx].failedAttempts 
             << "/" << MAX_ATTEMPTS << endl;
        if(users[idx].failedAttempts >= MAX_ATTEMPTS) {
            users[idx].locked = true;
            cout << "ACCOUNT LOCKED.\n" << endl;
        }
        saveUsers(users);
        return;
    }

    users[idx].failedAttempts = 0;
    saveUsers(users);
    isLoggedIn = true;
    isAdmin = false;
    currentUser = users[idx].username;

    cout << "\nLogin successful. Welcome, " << currentUser << "!\n" << endl;
}

void adminLogin() {
    cout << "\n========================================" << endl;
    cout << "          ADMIN LOGIN" << endl;
    cout << "========================================" << endl;

    string pass;
    cout << "\nEnter admin password: ";
    cin >> pass;

    if(pass != ADMIN_PASS) {
        cout << "\nInvalid admin password.\n" << endl;
        return;
    }

    isLoggedIn = true;
    isAdmin = true;
    currentUser = "ADMIN";
    cout << "\nAdmin login successful.\n" << endl;
}

void viewProfile() {
    if(!isLoggedIn || isAdmin) {
        cout << "Please login as user first.\n" << endl;
        return;
    }
    vector<User> users = loadUsers();
    for(size_t i = 0; i < users.size(); i++) {
        if(users[i].username == currentUser) {
            cout << "\n-------------- YOUR PROFILE ------------" << endl;
            cout << "Username: " << users[i].username << endl;
            cout << "Email:    " << users[i].email << endl;
            cout << "Joined:   " << users[i].regDate << endl;
            cout << "Status:   " << (users[i].locked ? "LOCKED" : "ACTIVE") << endl;
            cout << "----------------------------------------" << endl;
            return;
        }
    }
}

void changePassword() {
    if(!isLoggedIn || isAdmin) {
        cout << "Please login as user first.\n" << endl;
        return;
    }
    string oldPw, newPw, confirm;
    cout << "Current password: ";
    cin >> oldPw;

    vector<User> users = loadUsers();
    for(size_t i = 0; i < users.size(); i++) {
        if(users[i].username == currentUser) {
            if(hashPass(oldPw) != users[i].passwordHash) {
                cout << "Incorrect password.\n" << endl;
                return;
            }
            cout << "New password: ";
            cin >> newPw;
            while(!validPassword(newPw)) {
                cout << "Weak password. Try again: ";
                cin >> newPw;
            }
            cout << "Confirm new password: ";
            cin >> confirm;
            if(confirm != newPw) {
                cout << "Mismatch. Cancelled.\n" << endl;
                return;
            }
            users[i].passwordHash = hashPass(newPw);
            saveUsers(users);
            cout << "Password updated.\n" << endl;
            return;
        }
    }
}

void adminViewUsers() {
    if(!isLoggedIn || !isAdmin) {
        cout << "Admin access required.\n" << endl;
        return;
    }
    vector<User> users = loadUsers();
    cout << "\n========================================" << endl;
    cout << "         ALL REGISTERED USERS" << endl;
    cout << "========================================" << endl;
    if(users.empty()) {
        cout << "No users found.\n" << endl;
        return;
    }
    cout << "No  Username            Email                      Status" << endl;
    cout << "--------------------------------------------------------" << endl;
    for(size_t i = 0; i < users.size(); i++) {
        cout << left << setw(4) << (int)(i+1) 
             << setw(20) << users[i].username
             << setw(27) << users[i].email
             << (users[i].locked ? "LOCKED" : "ACTIVE") << endl;
    }
    cout << "--------------------------------------------------------" << endl;
    cout << "Total users: " << users.size() << "\n" << endl;
}

void adminUnlockUser() {
    if(!isLoggedIn || !isAdmin) {
        cout << "Admin access required.\n" << endl;
        return;
    }
    string name;
    cout << "Enter username to unlock: ";
    cin >> name;

    vector<User> users = loadUsers();
    bool found = false;
    for(size_t i = 0; i < users.size(); i++) {
        if(users[i].username == name) {
            users[i].locked = false;
            users[i].failedAttempts = 0;
            found = true;
            break;
        }
    }
    if(found) {
        saveUsers(users);
        cout << "User unlocked.\n" << endl;
    } else {
        cout << "User not found.\n" << endl;
    }
}

void logout() {
    if(!isLoggedIn) {
        cout << "Not logged in.\n" << endl;
        return;
    }
    cout << "Goodbye, " << currentUser << ".\n" << endl;
    isLoggedIn = false;
    isAdmin = false;
    currentUser = "";
}

void mainMenu() {
    int choice = 0;
    bool run = true;

    while(run) {
        cout << "\n========================================" << endl;
        cout << "   SECURE AUTHENTICATION SYSTEM" << endl;
        cout << "========================================" << endl;

        if(isLoggedIn) {
            cout << "  Logged in as: " << currentUser;
            if(isAdmin) cout << " [ADMIN]";
            cout << endl;
            cout << "----------------------------------------" << endl;
        }

        if(!isLoggedIn) {
            cout << "  1. Register New User" << endl;
            cout << "  2. User Login" << endl;
            cout << "  3. Admin Login" << endl;
            cout << "  0. Exit" << endl;
        } else if(isAdmin) {
            cout << "  4. View All Users" << endl;
            cout << "  5. Unlock User Account" << endl;
            cout << "  6. Logout" << endl;
            cout << "  0. Exit" << endl;
        } else {
            cout << "  4. View Profile" << endl;
            cout << "  5. Change Password" << endl;
            cout << "  6. Logout" << endl;
            cout << "  0. Exit" << endl;
        }

        cout << "----------------------------------------" << endl;
        cout << "Enter choice: ";
        cin >> choice;

        if(cin.fail()) {
            cin.clear();
            cin.ignore(1000, '\n');
            cout << "Invalid input.\n" << endl;
            continue;
        }

        switch(choice) {
            case 1: if(!isLoggedIn) registerUser(); else cout << "Invalid choice.\n"; break;
            case 2: if(!isLoggedIn) userLogin(); else cout << "Invalid choice.\n"; break;
            case 3: if(!isLoggedIn) adminLogin(); else cout << "Invalid choice.\n"; break;
            case 4: 
                if(isAdmin) adminViewUsers();
                else if(isLoggedIn) viewProfile();
                else cout << "Invalid choice.\n";
                break;
            case 5:
                if(isAdmin) adminUnlockUser();
                else if(isLoggedIn) changePassword();
                else cout << "Invalid choice.\n";
                break;
            case 6:
                if(isLoggedIn) logout();
                else cout << "Invalid choice.\n";
                break;
            case 0:
                run = false;
                break;
            default:
                cout << "Invalid choice.\n" << endl;
        }
    }
    cout << "\nSystem shutdown.\n";
}

int main() {
    mainMenu();
    return 0;
}
```
