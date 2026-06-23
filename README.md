#include<iostream>
#include<fstream>
#include<string>
#include<cstring>
using namespace std;

struct Book {
    int bookid;
    char title[60];
    char author[40];
    int totalcopies;
    int issuedcopies;
};

struct Member {
    int memberid;
    char membername[50];
    int bookid_issued;   // 0 means no book issued
    char issuedate[15];
    char returndate[15];
};

// ---- BOOK FUNCTIONS ----

void addBook() {
    Book b;
    ofstream file("books.dat", ios::binary | ios::app);

    cout << "\nEnter Book ID: ";
    cin >> b.bookid;
    cout << "Enter Title: ";
    cin.ignore();
    cin.getline(b.title, 60);
    cout << "Enter Author: ";
    cin.getline(b.author, 40);
    cout << "Enter Total Copies: ";
    cin >> b.totalcopies;
    b.issuedcopies = 0;

    file.write((char*)&b, sizeof(b));
    file.close();
    cout << "\nBook added!\n";
}

void displayBooks() {
    Book b;
    ifstream file("books.dat", ios::binary);

    if(!file) {
        cout << "\nNo books found!\n";
        return;
    }

    cout << "\n--- Book List ---\n";
    cout << "ID\tTitle\t\t\tAuthor\t\tAvailable\n";
    cout << "------------------------------------------------------\n";

    while(file.read((char*)&b, sizeof(b))) {
        int available = b.totalcopies - b.issuedcopies;
        cout << b.bookid << "\t" << b.title << "\t\t" << b.author << "\t\t" << available << "\n";
    }
    file.close();
}

void searchBook() {
    char keyword[60];
    Book b;
    ifstream file("books.dat", ios::binary);
    bool found = false;

    cout << "\nSearch by (1)Title or (2)Author? ";
    int opt;
    cin >> opt;
    cin.ignore();

    if(opt == 1) {
        cout << "Enter Title keyword: ";
        cin.getline(keyword, 60);
    } else {
        cout << "Enter Author name: ";
        cin.getline(keyword, 60);
    }

    cout << "\n--- Search Results ---\n";
    while(file.read((char*)&b, sizeof(b))) {
        string tocheck = (opt == 1) ? b.title : b.author;
        string kw = keyword;

        // simple check if keyword is in the title/author
        if(tocheck.find(kw) != string::npos) {
            cout << "ID: " << b.bookid << " | Title: " << b.title << " | Author: " << b.author;
            cout << " | Available: " << (b.totalcopies - b.issuedcopies) << "\n";
            found = true;
        }
    }
    file.close();

    if(!found) cout << "No matching books found!\n";
}

// ---- MEMBER FUNCTIONS ----

void addMember() {
    Member m;
    ofstream file("members.dat", ios::binary | ios::app);

    cout << "\nEnter Member ID: ";
    cin >> m.memberid;
    cout << "Enter Member Name: ";
    cin.ignore();
    cin.getline(m.membername, 50);
    m.bookid_issued = 0;
    strcpy(m.issuedate, "none");
    strcpy(m.returndate, "none");

    file.write((char*)&m, sizeof(m));
    file.close();
    cout << "\nMember added!\n";
}

void issueBook() {
    int memberid, bookid;
    cout << "\nEnter Member ID: ";
    cin >> memberid;
    cout << "Enter Book ID to issue: ";
    cin >> bookid;

    // update book issued count
    Book b;
    fstream bfile("books.dat", ios::binary | ios::in | ios::out);
    bool bookfound = false;
    bool bookavail = false;

    while(bfile.read((char*)&b, sizeof(b))) {
        if(b.bookid == bookid) {
            if(b.issuedcopies < b.totalcopies) {
                b.issuedcopies++;
                int pos = bfile.tellg();
                bfile.seekp(pos - sizeof(b));
                bfile.write((char*)&b, sizeof(b));
                bookavail = true;
            }
            bookfound = true;
            break;
        }
    }
    bfile.close();

    if(!bookfound) { cout << "\nBook not found!\n"; return; }
    if(!bookavail) { cout << "\nSorry, no copies available!\n"; return; }

    // update member record
    Member m;
    fstream mfile("members.dat", ios::binary | ios::in | ios::out);
    bool memberfound = false;

    while(mfile.read((char*)&m, sizeof(m))) {
        if(m.memberid == memberid) {
            if(m.bookid_issued != 0) {
                cout << "\nMember already has a book issued!\n";
                memberfound = true;
                break;
            }
            m.bookid_issued = bookid;

            cout << "Enter Issue Date (DD/MM/YYYY): ";
            cin.ignore();
            cin.getline(m.issuedate, 15);
            cout << "Enter Expected Return Date (DD/MM/YYYY): ";
            cin.getline(m.returndate, 15);

            int pos = mfile.tellg();
            mfile.seekp(pos - sizeof(m));
            mfile.write((char*)&m, sizeof(m));
            cout << "\nBook issued successfully!\n";
            memberfound = true;
            break;
        }
    }
    mfile.close();

    if(!memberfound) cout << "\nMember not found!\n";
}

void returnBook() {
    int memberid;
    cout << "\nEnter Member ID: ";
    cin >> memberid;

    Member m;
    fstream mfile("members.dat", ios::binary | ios::in | ios::out);
    bool found = false;

    while(mfile.read((char*)&m, sizeof(m))) {
        if(m.memberid == memberid) {
            if(m.bookid_issued == 0) {
                cout << "\nThis member has no book issued!\n";
                found = true;
                break;
            }

            int returnBookId = m.bookid_issued;

            // update book
            Book b;
            fstream bfile("books.dat", ios::binary | ios::in | ios::out);
            while(bfile.read((char*)&b, sizeof(b))) {
                if(b.bookid == returnBookId) {
                    b.issuedcopies--;
                    int bpos = bfile.tellg();
                    bfile.seekp(bpos - sizeof(b));
                    bfile.write((char*)&b, sizeof(b));
                    break;
                }
            }
            bfile.close();

            m.bookid_issued = 0;
            strcpy(m.issuedate, "none");
            strcpy(m.returndate, "none");

            int pos = mfile.tellg();
            mfile.seekp(pos - sizeof(m));
            mfile.write((char*)&m, sizeof(m));

            cout << "\nBook returned successfully!\n";
            found = true;
            break;
        }
    }
    mfile.close();

    if(!found) cout << "\nMember not found!\n";
}

void showMembers() {
    Member m;
    ifstream file("members.dat", ios::binary);

    if(!file) {
        cout << "\nNo members found!\n";
        return;
    }

    cout << "\n--- Member List ---\n";
    while(file.read((char*)&m, sizeof(m))) {
        cout << "ID: " << m.memberid << " | Name: " << m.membername;
        if(m.bookid_issued != 0)
            cout << " | Book Issued: " << m.bookid_issued << " | Issue: " << m.issuedate << " | Return By: " << m.returndate;
        else
            cout << " | No book issued";
        cout << "\n";
    }
    file.close();
}

int main() {
    int choice;
    cout << "=== Library Management System ===\n";

    do {
        cout << "\n--- BOOKS ---";
        cout << "\n1. Add Book";
        cout << "\n2. Display All Books";
        cout << "\n3. Search Book";
        cout << "\n--- MEMBERS ---";
        cout << "\n4. Add Member";
        cout << "\n5. Show All Members";
        cout << "\n--- ISSUE/RETURN ---";
        cout << "\n6. Issue Book";
        cout << "\n7. Return Book";
        cout << "\n8. Exit";
        cout << "\n\nEnter choice: ";
        cin >> choice;

        switch(choice) {
            case 1: addBook(); break;
            case 2: displayBooks(); break;
            case 3: searchBook(); break;
            case 4: addMember(); break;
            case 5: showMembers(); break;
            case 6: issueBook(); break;
            case 7: returnBook(); break;
            case 8: cout << "\nGoodbye!\n"; break;
            default: cout << "\nInvalid choice!\n";
        }
    } while(choice != 8);

    return 0;
}
