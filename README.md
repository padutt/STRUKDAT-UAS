#include <iostream>
#include <string>
#include <deque>
#include <stack>
#include <queue>
#include <vector>
#include <algorithm>

using namespace std;

// ==========================================
// 1. STRUKTUR DATA FAVORITES (Doubly Linked List)
// ==========================================
struct DLLNode {
    string name;
    string phone;
    DLLNode* prev;
    DLLNode* next;
    DLLNode(string n, string p) : name(n), phone(p), prev(nullptr), next(nullptr) {}
};

class DoublyLinkedList {
private:
    DLLNode* head;
public:
    DoublyLinkedList() : head(nullptr) {}
    
    void add_favorite(string name, string phone) {
        DLLNode* newNode = new DLLNode(name, phone);
        if (!head) {
            head = newNode;
        } else {
            DLLNode* curr = head;
            while (curr->next) curr = curr->next;
            curr->next = newNode;
            newNode->prev = curr;
        }
        cout << "[" << name << "] ditambahkan ke Favorit.\n";
    }
};

// ==========================================
// 2. BUKU TELEPON UTAMA (AVL Tree)
// ==========================================
struct AVLNode {
    string name;
    string phone;
    AVLNode* left;
    AVLNode* right;
    int height;
    
    AVLNode(string n, string p) : name(n), phone(p), left(nullptr), right(nullptr), height(1) {}
};

class AVLTree {
public:
    int getHeight(AVLNode* node) {
        if (!node) return 0;
        return node->height;
    }

    int getBalance(AVLNode* node) {
        if (!node) return 0;
        return getHeight(node->left) - getHeight(node->right);
    }

    AVLNode* rightRotate(AVLNode* y) {
        AVLNode* x = y->left;
        AVLNode* T2 = x->right;
        x->right = y;
        y->left = T2;
        y->height = max(getHeight(y->left), getHeight(y->right)) + 1;
        x->height = max(getHeight(x->left), getHeight(x->right)) + 1;
        return x;
    }

    AVLNode* leftRotate(AVLNode* x) {
        AVLNode* y = x->right;
        AVLNode* T2 = y->left;
        y->left = x;
        x->right = T2;
        x->height = max(getHeight(x->left), getHeight(x->right)) + 1;
        y->height = max(getHeight(y->left), getHeight(y->right)) + 1;
        return y;
    }

    AVLNode* insertNode(AVLNode* node, string name, string phone) {
        if (!node) return new AVLNode(name, phone);

        if (name < node->name)
            node->left = insertNode(node->left, name, phone);
        else if (name > node->name)
            node->right = insertNode(node->right, name, phone);
        else
            return node; 

        node->height = 1 + max(getHeight(node->left), getHeight(node->right));
        int balance = getBalance(node);

        // Rotasi untuk Balancing
        if (balance > 1 && name < node->left->name) return rightRotate(node);
        if (balance < -1 && name > node->right->name) return leftRotate(node);
        if (balance > 1 && name > node->left->name) {
            node->left = leftRotate(node->left);
            return rightRotate(node);
        }
        if (balance < -1 && name < node->right->name) {
            node->right = rightRotate(node->right);
            return leftRotate(node);
        }
        return node;
    }

    AVLNode* searchNode(AVLNode* root, string name) {
        if (root == nullptr || root->name == name) return root;
        if (root->name < name) return searchNode(root->right, name);
        return searchNode(root->left, name);
    }

    AVLNode* minValueNode(AVLNode* node) {
        AVLNode* current = node;
        while (current->left != nullptr) current = current->left;
        return current;
    }

    AVLNode* deleteNode(AVLNode* root, string name) {
        if (!root) return root;

        if (name < root->name) root->left = deleteNode(root->left, name);
        else if (name > root->name) root->right = deleteNode(root->right, name);
        else {
            if (!root->left || !root->right) {
                AVLNode* temp = root->left ? root->left : root->right;
                if (!temp) {
                    temp = root;
                    root = nullptr;
                } else {
                    *root = *temp; 
                }
                delete temp;
            } else {
                AVLNode* temp = minValueNode(root->right);
                root->name = temp->name;
                root->phone = temp->phone;
                root->right = deleteNode(root->right, temp->name);
            }
        }

        if (!root) return root;

        root->height = 1 + max(getHeight(root->left), getHeight(root->right));
        int balance = getBalance(root);

        if (balance > 1 && getBalance(root->left) >= 0) return rightRotate(root);
        if (balance > 1 && getBalance(root->left) < 0) {
            root->left = leftRotate(root->left);
            return rightRotate(root);
        }
        if (balance < -1 && getBalance(root->right) <= 0) return leftRotate(root);
        if (balance < -1 && getBalance(root->right) > 0) {
            root->right = rightRotate(root->right);
            return leftRotate(root);
        }
        return root;
    }
};

// ==========================================
// 3. SISTEM MANAJEMEN KONTAK TERPADU
// ==========================================

// Struct untuk mempermudah penyimpanan di Stack & Priority Queue
struct ContactData {
    string name;
    string phone;
};

struct CallQueueItem {
    int priority;
    string name;
    // Overload operator < untuk Max-Heap di Priority Queue
    bool operator<(const CallQueueItem& other) const {
        return priority < other.priority; 
    }
};

class ContactManager {
private:
    AVLTree avl;
    AVLNode* root;
    
    deque<string> recent_searches;        // Deque untuk Riwayat
    DoublyLinkedList favorites;           // DLL untuk Favorit
    stack<ContactData> undo_stack;        // Stack untuk Undo Hapus
    priority_queue<CallQueueItem> call_q; // Priority Queue untuk Telepon

public:
    ContactManager() : root(nullptr) {}

    void add_favorite(string name, string phone) {
        favorites.add_favorite(name, phone);
    }

    void add_contact(string name, string phone) {
        root = avl.insertNode(root, name, phone);
        cout << "Kontak [" << name << "] berhasil ditambahkan ke Direktori.\n";
    }

    // Fitur 1: Recent Searches
    void search_contact(string name) {
        AVLNode* result = avl.searchNode(root, name);
        if (result) {
            // Hapus riwayat jika nama sudah ada agar data tetap unik
            auto it = find(recent_searches.begin(), recent_searches.end(), name);
            if (it != recent_searches.end()) {
                recent_searches.erase(it);
            }
            
            // Masukkan ke posisi depan
            recent_searches.push_front(name);
            
            // Batasi kapasitas riwayat (misal 10 data maksimal)
            if (recent_searches.size() > 10) {
                recent_searches.pop_back();
            }
            
            cout << "Ditemukan: " << result->name << " - " << result->phone << "\n";
        } else {
            cout << "Kontak " << name << " tidak ditemukan.\n";
        }
    }

    void show_recent_searches() {
        cout << "Recent Searches: ";
        for (const auto& s : recent_searches) {
            cout << s << " | ";
        }
        cout << "\n";
    }

    // Fitur 3: Delete & Undo 
    void delete_contact(string name) {
        AVLNode* node = avl.searchNode(root, name);
        if (node) {
            // Simpan ke stack sebelum dihapus dari Tree
            undo_stack.push({node->name, node->phone});
            root = avl.deleteNode(root, name);
            cout << "Kontak [" << name << "] dihapus (Bisa di-Undo).\n";
        } else {
            cout << "Kontak tidak ditemukan untuk dihapus.\n";
        }
    }

    void undo_delete() {
        if (!undo_stack.empty()) {
            ContactData last_deleted = undo_stack.top();
            undo_stack.pop();
            add_contact(last_deleted.name, last_deleted.phone);
            cout << "Undo berhasil dilakukan.\n";
        } else {
            cout << "Tidak ada aksi yang bisa di-Undo.\n";
        }
    }

    // Fitur 4: Call Queue
    void enqueue_call(string name, int priority) {
        call_q.push({priority, name});
        cout << "Menambahkan [" << name << "] ke Antrean Panggilan dengan Prioritas " << priority << ".\n";
    }

    void call_next() {
        if (!call_q.empty()) {
            CallQueueItem item = call_q.top();
            call_q.pop();
            cout << "Memanggil: " << item.name << " (Prioritas " << item.priority << ")...\n";
        } else {
            cout << "Antrean panggilan kosong.\n";
        }
    }
};

// ==========================================
// 4. MAIN FUNCTION (Simulasi)
// ==========================================
int main() {
    ContactManager app;
    int pilihan;
    string nama, nomor;
    int prioritas;

    // Data awal (Dummy data) agar aplikasi tidak kosong saat pertama dibuka
    app.add_contact("Budi", "0812345678");
    app.add_contact("Andi", "0813999999");
    app.add_contact("Zaki", "0814111111");
    system("clear || cls"); // Membersihkan layar terminal awal

    while (true) {
        cout << "\n===============================================\n";
        cout << "    APLIKASI MANAJEMEN KONTAK (PROJECT UAS)    \n";
        cout << "===============================================\n";
        cout << "1. Tambah Kontak Baru (AVL Tree)\n";
        cout << "2. Cari Kontak (Recent Searches - Deque)\n";
        cout << "3. Tambah ke Kontak Favorit (Doubly Linked List)\n";
        cout << "4. Hapus Kontak (Undo System - Stack)\n";
        cout << "5. Undo Hapus Kontak (LIFO Stack)\n";
        cout << "6. Masukkan ke Antrean Panggilan (Priority Queue)\n";
        cout << "7. Panggil Kontak Berikutnya (Proses Antrean)\n";
        cout << "8. Lihat Riwayat Pencarian Terbaru\n";
        cout << "9. Keluar Aplikasi\n";
        cout << "===============================================\n";
        cout << "Pilih menu (1-9): ";
        cin >> pilihan;

        // Validasi input agar tidak error jika user salah ketik huruf
        if (cin.fail()) {
            cin.clear();
            cin.ignore(10000, '\n');
            cout << "Input tidak valid! Masukkan angka.\n";
            continue;
        }

        switch (pilihan) {
            case 1:
                cout << "\n--- TAMBAH KONTAK ---\n";
                cout << "Masukkan Nama: "; cin >> nama;
                cout << "Masukkan Nomor HP: "; cin >> nomor;
                app.add_contact(nama, nomor);
                break;

            case 2:
                cout << "\n--- CARI KONTAK ---\n";
                cout << "Masukkan Nama yang dicari: "; cin >> nama;
                app.search_contact(nama); // Menggunakan Deque [cite: 6]
                break;

            case 3:
                cout << "\n--- TAMBAH FAVORIT ---\n";
                cout << "Masukkan Nama Kontak: "; cin >> nama;
                cout << "Masukkan Nomor HP: "; cin >> nomor;
                app.add_favorite(nama, nomor); // Menggunakan Doubly Linked List [cite: 17]
                break;

          case 4:
                cout << "\n--- HAPUS KONTAK ---\n";
                cout << "Masukkan Nama yang akan dihapus: "; cin >> nama;
                app.delete_contact(nama); 
                break;

            case 5:
                cout << "\n--- UNDO HAPUS KONTAK ---\n";
                app.undo_delete(); // Pop dari Stack [cite: 29]
                break;

            case 6:
                cout << "\n--- MASUKKAN ANTREAN PANGGILAN ---\n";
                cout << "Masukkan Nama Kontak: "; cin >> nama;
                cout << "Kategori Prioritas:\n";
                cout << " 3 = Emergency / Keluarga (Tinggi)\n";
                cout << " 2 = Teman Dekat (Sedang)\n";
                cout << " 1 = Kontak Lainnya (Rendah)\n";
                cout << "Pilih Bobot (1-3): "; cin >> prioritas; // Sesuai tabel dokumen 
                if (prioritas >= 1 && prioritas <= 3) {
                    app.enqueue_call(nama, prioritas); // Menggunakan Priority Queue [cite: 32]
                } else {
                    cout << "Prioritas tidak valid!\n";
                }
                break;

            case 7:
                cout << "\n--- PROSES PANGGILAN ---\n";
                app.call_next(); 
                break;

            case 8:
                cout << "\n--- RIWAYAT PENCARIAN TERBARU ---\n";
                app.show_recent_searches(); // Menampilkan isi Deque [cite: 4]
                break;

            case 9:
                cout << "\nTerima kasih! Aplikasi ditutup.\n";
                return 0;

            default:
                cout << "Pilihan tidak tersedia. Silakan coba lagi.\n";
        }
    }
    return 0;
}
