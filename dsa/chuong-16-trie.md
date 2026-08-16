# Chương 16: Trie (Cây tiền tố)

## 16.1. Khái niệm cốt lõi

### 16.1.1. Định nghĩa

Trie (còn gọi là Prefix Tree — cây tiền tố) là cấu trúc dữ liệu dạng cây chuyên biệt để lưu trữ một tập hợp chuỗi, trong đó mỗi **cạnh** biểu diễn một ký tự, và đường đi từ root đến một node bất kỳ biểu diễn một **tiền tố (prefix)** của các chuỗi đã lưu. Các chuỗi có chung tiền tố sẽ **dùng chung đường đi** từ root cho đến điểm chúng bắt đầu khác nhau — đây chính là bản chất giúp Trie tiết kiệm bộ nhớ và tăng tốc truy vấn theo tiền tố so với lưu trữ từng chuỗi độc lập trong HashSet.

### 16.1.2. Bản chất — vì sao Trie nhanh hơn HashSet cho truy vấn tiền tố

Với HashSet lưu trữ tập hợp chuỗi (mục 3.2), việc kiểm tra "có tồn tại chuỗi nào bắt đầu bằng tiền tố X hay không" đòi hỏi duyệt qua **toàn bộ** các chuỗi trong tập hợp và so sánh từng chuỗi với tiền tố X — độ phức tạp O(n · L) với `n` là số chuỗi, `L` là độ dài tiền tố. Trie giải quyết vấn đề này triệt để: vì cấu trúc cây được tổ chức theo đúng từng ký tự của tiền tố, việc kiểm tra tồn tại tiền tố X chỉ cần đi theo đúng X ký tự từ root — độ phức tạp O(L), **hoàn toàn không phụ thuộc vào số lượng chuỗi** `n` đã lưu trong Trie.

**Minh họa cấu trúc Trie** lưu trữ ba từ `"cat"`, `"car"`, `"dog"`:

```
                    root
                   /    \
                  c      d
                  |      |
                  a      o
                 / \     |
                t   r    g
                *   *    *      ← dấu * đánh dấu kết thúc một từ hoàn chỉnh (isEndOfWord)
```

Quan sát quan trọng: `"cat"` và `"car"` **dùng chung** đường đi `c → a`, chỉ tách nhánh tại ký tự thứ ba — đây chính là cách Trie tiết kiệm bộ nhớ khi các chuỗi có tiền tố chung, và cũng là lý do một truy vấn "liệt kê mọi từ bắt đầu bằng `ca`" chỉ cần định vị node `a` (qua 2 bước) rồi khám phá toàn bộ cây con bên dưới nó, thay vì quét lại từ đầu toàn bộ tập dữ liệu.

### 16.1.3. Cấu trúc node

Mỗi node trong Trie cần lưu: một mảng/bảng con trỏ đến các node con (thường có kích thước cố định bằng bảng chữ cái, ví dụ 26 cho chữ cái tiếng Anh thường), và một cờ đánh dấu **node này có phải là điểm kết thúc của một từ hoàn chỉnh hay không** — vì một tiền tố hợp lệ chưa chắc là một từ hoàn chỉnh đã được chèn (ví dụ `"ca"` là tiền tố hợp lệ của `"cat"` nhưng bản thân `"ca"` có thể chưa từng được chèn như một từ độc lập).

```cpp
struct TrieNode {
    TrieNode* children[26]; // mỗi ô ứng với một chữ cái 'a' đến 'z'
    bool isEndOfWord;

    TrieNode() : isEndOfWord(false) {
        for (int i = 0; i < 26; i++) children[i] = nullptr;
    }
};
```

---

## 16.2. Các thao tác cơ bản

### 16.2.1. Insert (Chèn từ)

**Bản chất:** đi theo từng ký tự của từ cần chèn, tại mỗi bước nếu con trỏ tương ứng chưa tồn tại thì tạo node mới; sau khi đi hết toàn bộ từ, đánh dấu node cuối cùng là điểm kết thúc từ (`isEndOfWord = true`).

```cpp
class Trie {
private:
    TrieNode* root;

public:
    Trie() {
        root = new TrieNode();
    }

    void insert(const string& word) {
        TrieNode* curr = root;

        for (char c : word) {
            int index = c - 'a';
            if (curr->children[index] == nullptr) {
                curr->children[index] = new TrieNode(); // tạo nhánh mới nếu chưa có
            }
            curr = curr->children[index];
        }

        curr->isEndOfWord = true; // đánh dấu điểm kết thúc từ hoàn chỉnh
    }
```

**Độ phức tạp:** O(L) thời gian với `L` là độ dài từ cần chèn, O(L) bộ nhớ phụ trong trường hợp xấu nhất (khi không có ký tự nào trùng với các từ đã chèn trước đó).

### 16.2.2. Search (Tìm kiếm từ hoàn chỉnh)

**Bản chất:** đi theo từng ký tự của từ cần tìm; nếu tại bất kỳ bước nào con trỏ tương ứng không tồn tại, từ chắc chắn chưa được chèn — trả về false ngay lập tức. Nếu đi hết toàn bộ từ mà vẫn còn trong cây, **phải kiểm tra thêm cờ `isEndOfWord`** tại node cuối — vì có thể toàn bộ đường đi tồn tại (do là tiền tố của một từ dài hơn đã chèn) nhưng bản thân từ đang tìm chưa từng được chèn như một từ độc lập.

```cpp
    bool search(const string& word) {
        TrieNode* node = findNode(word);
        return node != nullptr && node->isEndOfWord;
    }

    bool startsWith(const string& prefix) {
        return findNode(prefix) != nullptr; // chỉ cần tồn tại đường đi, KHÔNG cần isEndOfWord
    }

private:
    TrieNode* findNode(const string& s) {
        TrieNode* curr = root;

        for (char c : s) {
            int index = c - 'a';
            if (curr->children[index] == nullptr) return nullptr; // đường đi bị gãy giữa chừng
            curr = curr->children[index];
        }

        return curr;
    }
};
```

**Giải thích khác biệt then chốt giữa `search` và `startsWith`:** `search` yêu cầu chuỗi phải là một từ hoàn chỉnh đã được chèn (kiểm tra `isEndOfWord`), trong khi `startsWith` chỉ cần xác nhận **tồn tại đường đi** tương ứng trong cây, không quan tâm node cuối có phải điểm kết thúc từ hay không — đây chính là bản chất "tra cứu tiền tố" mà HashSet không thể làm hiệu quả (mục 16.1.2).

**Độ phức tạp (cả hai hàm):** O(L) thời gian với `L` là độ dài chuỗi truy vấn — không phụ thuộc số lượng từ đã lưu trong Trie.

---

## 16.3. So sánh Trie với HashSet cho bài toán lưu trữ chuỗi

| Tiêu chí | HashSet | Trie |
|---|---|---|
| Kiểm tra tồn tại từ hoàn chỉnh | O(L) trung bình (L = độ dài chuỗi, tính hash) | O(L) |
| Kiểm tra tồn tại tiền tố | O(n · L) — phải duyệt và so sánh từng chuỗi | O(L) — không phụ thuộc n |
| Liệt kê mọi từ có chung tiền tố | O(n · L) | O(L + số lượng kết quả) |
| Bộ nhớ khi nhiều chuỗi chung tiền tố | O(n · L) — mỗi chuỗi lưu độc lập | Tiết kiệm hơn nhờ dùng chung đường đi |
| Bộ nhớ khi các chuỗi ít chung tiền tố | Hiệu quả hơn (không có overhead con trỏ mỗi ký tự) | Có thể tốn kém hơn do overhead cấu trúc node |

**Khi nào dùng Trie:** khi bài toán liên quan trực tiếp đến **tiền tố** — tính năng autocomplete/gợi ý từ, kiểm tra chính tả, tìm kiếm từ điển theo tiền tố, hoặc bài toán yêu cầu liệt kê mọi chuỗi thỏa mãn một mẫu tiền tố nhất định. Nếu bài toán chỉ cần kiểm tra tồn tại chuỗi hoàn chỉnh (không liên quan tiền tố), HashSet thường đơn giản và đủ hiệu quả hơn.

---

## 16.4. Cài đặt bài toán mở rộng

### 16.4.1. Word Search II (kết hợp Trie với Backtracking)

**Bài toán:** cho lưới ký tự 2D và một danh sách từ, tìm mọi từ trong danh sách có thể tạo thành trên lưới (tương tự Word Search ở mục 13.3.6, nhưng với **nhiều từ cùng lúc** thay vì một từ).

**Bản chất kết hợp:** nếu áp dụng trực tiếp thuật toán Word Search (mục 13.3.6) riêng lẻ cho từng từ trong danh sách, độ phức tạp sẽ nhân thêm một hệ số bằng số lượng từ — rất lãng phí vì nhiều từ có thể **chung tiền tố**, dẫn đến việc khám phá lặp lại cùng một phần của lưới nhiều lần. Giải pháp: gộp toàn bộ danh sách từ vào một Trie duy nhất, sau đó chỉ thực hiện **một lượt Backtracking duy nhất** trên lưới — tại mỗi bước, dùng Trie để kiểm tra xem đường đi hiện tại có còn là tiền tố hợp lệ của **bất kỳ** từ nào trong danh sách hay không (thay vì kiểm tra riêng cho từng từ), cho phép Pruning (mục 13.1.4) sớm và chia sẻ công việc khám phá giữa các từ có chung tiền tố.

```cpp
#include <vector>
#include <string>
using namespace std;

struct TrieNodeWS {
    TrieNodeWS* children[26] = {};
    string word = ""; // lưu trực tiếp từ hoàn chỉnh tại node kết thúc, thay vì chỉ cờ boolean
};

void insertWord(TrieNodeWS* root, const string& word) {
    TrieNodeWS* curr = root;
    for (char c : word) {
        int idx = c - 'a';
        if (curr->children[idx] == nullptr) curr->children[idx] = new TrieNodeWS();
        curr = curr->children[idx];
    }
    curr->word = word;
}

void backtrackWordSearch(vector<vector<char>>& board, int row, int col,
                          TrieNodeWS* node, vector<string>& result) {
    if (row < 0 || row >= (int)board.size() || col < 0 || col >= (int)board[0].size()) return;

    char c = board[row][col];
    if (c == '#' || node->children[c - 'a'] == nullptr) return; // Pruning qua Trie

    TrieNodeWS* nextNode = node->children[c - 'a'];
    if (!nextNode->word.empty()) {
        result.push_back(nextNode->word);
        nextNode->word = ""; // tránh thêm trùng lặp nếu tìm thấy lại từ này qua đường khác
    }

    board[row][col] = '#'; // đánh dấu đã dùng trong đường đi hiện tại

    backtrackWordSearch(board, row + 1, col, nextNode, result);
    backtrackWordSearch(board, row - 1, col, nextNode, result);
    backtrackWordSearch(board, row, col + 1, nextNode, result);
    backtrackWordSearch(board, row, col - 1, nextNode, result);

    board[row][col] = c; // hoàn tác
}

vector<string> findWords(vector<vector<char>>& board, vector<string>& words) {
    TrieNodeWS* root = new TrieNodeWS();
    for (const string& w : words) insertWord(root, w);

    vector<string> result;
    for (int r = 0; r < (int)board.size(); r++) {
        for (int c = 0; c < (int)board[0].size(); c++) {
            backtrackWordSearch(board, r, c, root, result);
        }
    }

    return result;
}
```

**Độ phức tạp:** O(rows · cols · 4^L) thời gian trong trường hợp xấu nhất với `L` là độ dài từ dài nhất — về mặt lý thuyết cùng bậc với thực hiện Word Search riêng lẻ cho từng từ, nhưng **hiệu quả thực tế tốt hơn đáng kể** nhờ Pruning sớm khi đường đi không còn là tiền tố hợp lệ của bất kỳ từ nào, và nhờ chia sẻ công việc khám phá giữa các từ chung tiền tố — minh chứng cụ thể cho nguyên lý Pruning đã nêu ở mục 13.1.4 (không đổi độ phức tạp lý thuyết nhưng cải thiện hiệu năng thực tế).

---

## 16.5. Bảng tổng hợp

| Thao tác | Độ phức tạp | Ghi chú |
|---|---|---|
| Insert | O(L) | L = độ dài chuỗi chèn |
| Search (từ hoàn chỉnh) | O(L) | Cần kiểm tra `isEndOfWord` |
| StartsWith (tiền tố) | O(L) | Không cần kiểm tra `isEndOfWord` |
| Word Search II (Trie + Backtracking) | O(rows · cols · 4^L) lý thuyết | Hiệu quả thực tế tốt hơn nhờ Pruning chia sẻ |

---

## 16.6. Danh sách bài tập luyện tập

### Mức Easy
1. Implement Trie (Prefix Tree) — cài đặt lại từ đầu theo mục 16.2
2. Longest Common Prefix (thử giải lại bằng Trie, so sánh với cách giải ở mục 2.4.2)

### Mức Medium
3. Design Add and Search Words Data Structure (hỗ trợ ký tự đại diện `.` khớp mọi ký tự — cần Backtracking trên Trie)
4. Replace Words (thay từ bằng tiền tố ngắn nhất khớp trong từ điển)
5. Map Sum Pairs
6. Word Break (kết hợp Trie + Dynamic Programming — xem Chương 28)

### Mức Hard
7. Word Search II
8. Palindrome Pairs
9. Stream of Characters

---

*Chương tiếp theo: **Chương 17 — Graph Fundamentals**, mở rộng cấu trúc cây (vốn không cho phép chu trình và mỗi node chỉ có một cha) sang cấu trúc đồ thị tổng quát, nơi các thuật toán duyệt BFS/DFS đã manh nha xuất hiện ở Chương 8 và Chương 12-13 được hệ thống hóa đầy đủ.*
