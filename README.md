#ifndef CONTACTMANAGER_H
#define CONTACTMANAGER_H

#include <string>
#include <vector>
#include <deque>
#include <stack>
#include <queue>
#include <algorithm>

using namespace std;

// Struct pembantu untuk melempar data ke GUI Qt
struct ContactData {
    string name;
    string phone;
};

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
    }

    vector<ContactData> getAllFavorites() {
        vector<ContactData> res;
        DLLNode* curr = head;
        while (curr != nullptr) {
            res.push_back({curr->name, curr->phone});
            curr = curr->next;
        }
        return res;
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
    int getHeight(AVLNode* node) { return node ? node->height : 0; }
    int getBalance(AVLNode* node) { return node ? getHeight(node->left) - getHeight(node->right) : 0; }

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
        if (name < node->name) node->left = insertNode(node->left, name, phone);
        else if (name > node->name) node->right = insertNode(node->right, name, phone);
        else return node;

        node->height = 1 + max(getHeight(node->left), getHeight(node->right));
        int balance = getBalance(node);

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
        if (!root || root->name == name) return root;
        if (root->name < name) return searchNode(root->right, name);
        return searchNode(root->left, name);
    }

    AVLNode* minValueNode(AVLNode* node) {
        AVLNode* current = node;
        while (current && current->left != nullptr) current = current->left;
        return current;
    }

    AVLNode* deleteNode(AVLNode* root, string name) {
        if (!root) return root;
        if (name < root->name) root->left = deleteNode(root->left, name);
        else if (name > root->name) root->right = deleteNode(root->right, name);
        else {
            if (!root->left || !root->right) {
                AVLNode* temp = root->left ? root->left : root->right;
                if (!temp) { temp = root; root = nullptr; }
                else *root = *temp;
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
            root->left = leftRotate(root->left); // Fix contextual typo if any
            return rightRotate(root);
        }
        if (balance < -1 && getBalance(root->right) <= 0) return leftRotate(root);
        if (balance < -1 && getBalance(root->right) > 0) {
            root->right = rightRotate(root->right);
            return leftRotate(root);
        }
        return root;
    }

    // Fungsi tambahan untuk mengambil seluruh isi pohon secara terurut (In-Order)
    void getInOrder(AVLNode* node, vector<ContactData>& vec) {
        if (!node) return;
        getInOrder(node->left, vec);
        vec.push_back({node->name, node->phone});
        getInOrder(node->right, vec);
    }
};

// ==========================================
// 3. ANTREAN TELEPON (Priority Queue Item)
// ==========================================
struct CallQueueItem {
    int priority;
    string name;
    bool operator<(const CallQueueItem& other) const {
        return priority < other.priority;
    }
};

// ==========================================
// 4. MANAGER PENGHUBUNG UTAMA
// ==========================================
class ContactManager {
private:
    AVLTree avl;
    AVLNode* root;
    deque<string> recent_searches;
    DoublyLinkedList favorites;
    stack<ContactData> undo_stack;
    priority_queue<CallQueueItem> call_q;

public:
    ContactManager() : root(nullptr) {}

    void add_contact(string name, string phone) { root = avl.insertNode(root, name, phone); }
    void add_favorite(string name, string phone) { favorites.add_favorite(name, phone); }

    bool search_contact(string name, string& outPhone) {
        AVLNode* result = avl.searchNode(root, name);
        if (result) {
            auto it = find(recent_searches.begin(), recent_searches.end(), name);
            if (it != recent_searches.end()) recent_searches.erase(it);
            recent_searches.push_front(name);
            if (recent_searches.size() > 10) recent_searches.pop_back();
            outPhone = result->phone;
            return true;
        }
        return false;
    }

    bool delete_contact(string name) {
        AVLNode* node = avl.searchNode(root, name);
        if (node) {
            undo_stack.push({node->name, node->phone});
            root = avl.deleteNode(root, name);
            return true;
        }
        return false;
    }

    bool undo_delete() {
        if (!undo_stack.empty()) {
            ContactData last = undo_stack.top();
            undo_stack.pop();
            add_contact(last.name, last.phone);
            return true;
        }
        return false;
    }

    void enqueue_call(string name, int priority) { call_q.push({priority, name}); }

    bool call_next(CallQueueItem& outItem) {
        if (!call_q.empty()) {
            outItem = call_q.top();
            call_q.pop();
            return true;
        }
        return false;
    }

    // Fungsi jembatan data khusus untuk GUI Qt Creator
    vector<ContactData> get_all_contacts() {
        vector<ContactData> vec;
        avl.getInOrder(root, vec);
        return vec;
    }
    vector<string> get_recent_searches() { return vector<string>(recent_searches.begin(), recent_searches.end()); }
    vector<ContactData> get_favorites() { return favorites.getAllFavorites(); }
    vector<CallQueueItem> get_call_queue() {
        priority_queue<CallQueueItem> temp = call_q;
        vector<CallQueueItem> vec;
        while (!temp.empty()) {
            vec.push_back(temp.top());
            temp.pop();
        }
        return vec;
    }
};

#endif // CONTACTMANAGER_H
