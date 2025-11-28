#define _CRT_SECURE_NO_WARNINGS

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

// --- 1. 정의 및 전역 변수 설정 ---
#define MAX_RECORDS 200000 
#define MAX_NAME_LEN 50
#define MAX_LINE_LEN 200
#define REPETITIONS 1
#define FILENAME "students.csv"

// 측정 변수
long long comparisons = 0;
long long moves = 0;

typedef struct {
    int id;
    char name[MAX_NAME_LEN];
    char gender;
    int korean;
    int english;
    int math;
    int total;
} Student;

typedef int (*Comparator)(const Student*, const Student*);

// --- 2. 데이터 로드 및 헬퍼 함수 ---

Student* load_students(const char* filename, int* out_count) {
    FILE* fp = fopen(filename, "r");
    if (!fp) {
        perror("Failed to open file");
        return NULL;
    }

    char line[MAX_LINE_LEN];
    int capacity = 10000;
    int count = 0;
    Student* arr = (Student*)malloc(sizeof(Student) * capacity);

    if (!arr) {
        perror("Memory allocation failed");
        fclose(fp);
        return NULL;
    }

    if (fgets(line, sizeof(line), fp) == NULL) {
        fclose(fp);
        free(arr);
        *out_count = 0;
        return NULL;
    }

    while (fgets(line, sizeof(line), fp)) {
        if (count >= capacity) {
            capacity *= 2;
            Student* temp = (Student*)realloc(arr, sizeof(Student) * capacity);
            if (!temp) {
                fprintf(stderr, "Warning: Reallocation failed, returning partial data.\n");
                fclose(fp);
                *out_count = count;
                return arr;
            }
            arr = temp;
        }

        Student s;
        char temp_line[MAX_LINE_LEN];
        strncpy(temp_line, line, MAX_LINE_LEN - 1);
        temp_line[MAX_LINE_LEN - 1] = '\0';

        char* token = strtok(temp_line, ",");
        if (!token) break;
        s.id = atoi(token);

        token = strtok(NULL, ",");
        if (!token) break;
        strncpy(s.name, token, MAX_NAME_LEN - 1);
        s.name[MAX_NAME_LEN - 1] = '\0';

        token = strtok(NULL, ",");
        if (!token) break;
        s.gender = token[0];

        token = strtok(NULL, ",");
        if (!token) break;
        s.korean = atoi(token);

        token = strtok(NULL, ",");
        if (!token) break;
        s.english = atoi(token);

        token = strtok(NULL, ",");
        if (!token) break;
        s.math = atoi(token);

        s.total = s.korean + s.english + s.math;

        arr[count++] = s;
    }

    fclose(fp);

    Student* tight = (Student*)realloc(arr, sizeof(Student) * count);
    if (tight) {
        arr = tight;
    }
    else {
        fprintf(stderr, "Warning: Tight reallocation failed, using original memory.\n");
    }

    *out_count = count;
    return arr;
}

void swap_students(Student* a, Student* b) {
    Student temp = *a;
    *a = *b;
    *b = temp;
    moves += 3;
}

void shuffle_students(Student arr[], int n) {
    if (n > 1) {
        // srand는 main에서 한 번만 호출
        for (int i = n - 1; i > 0; i--) {
            int j = rand() % (i + 1);
            Student temp = arr[i];
            arr[i] = arr[j];
            arr[j] = temp;
        }
    }
}

// --- 3. 비교 함수 ---
int compare_id_asc(const Student* a, const Student* b) { comparisons++; return a->id - b->id; }
int compare_id_desc(const Student* a, const Student* b) { comparisons++; return b->id - a->id; }
int compare_name_asc(const Student* a, const Student* b) { comparisons++; return strcmp(a->name, b->name); }
int compare_name_desc(const Student* a, const Student* b) { comparisons++; return strcmp(b->name, a->name); }
int compare_gender_asc(const Student* a, const Student* b) { comparisons++; return a->gender - b->gender; }
int compare_gender_desc(const Student* a, const Student* b) { comparisons++; return b->gender - a->gender; }
int compare_total_custom(const Student* a, const Student* b) {
    comparisons++;
    if (a->total != b->total) { return a->total - b->total; }
    if (a->korean != b->korean) { return b->korean - a->korean; }
    if (a->english != b->english) { return b->english - a->english; }
    return b->math - a->math;
}
int compare_total_asc(const Student* a, const Student* b) { return compare_total_custom(a, b); }
int compare_total_desc(const Student* a, const Student* b) { return compare_total_custom(b, a); }


// --- 4. 최적화된 정렬 알고리즘 구현 ---

// 4.1. 셸 정렬 (Shell Sort) - Marcin Ciura의 최적화 간격 사용
void shell_sort(Student arr[], int n, Comparator cmp) {
    int gaps[] = { 1750, 701, 301, 132, 57, 23, 10, 4, 1 };
    int num_gaps = sizeof(gaps) / sizeof(gaps[0]);

    for (int g = 0; g < num_gaps; g++) {
        int gap = gaps[g];
        if (gap >= n) continue;

        for (int i = gap; i < n; i++) {
            Student temp = arr[i];
            moves++;
            int j;
            for (j = i; j >= gap; j -= gap) {
                if (cmp(&arr[j - gap], &temp) > 0) {
                    arr[j] = arr[j - gap];
                    moves++;
                }
                else {
                    break;
                }
            }
            arr[j] = temp;
            moves++;
        }
    }
}

// 4.2. 퀵 정렬 (Quick Sort) - 중앙값 피벗 (Median-of-Three) 최적화
int partition(Student arr[], int low, int high, Comparator cmp) {
    Student pivot = arr[high];
    moves++;
    int i = (low - 1);

    for (int j = low; j <= high - 1; j++) {
        if (cmp(&arr[j], &pivot) < 0) {
            i++;
            swap_students(&arr[i], &arr[j]);
        }
    }
    swap_students(&arr[i + 1], &arr[high]);
    return (i + 1);
}

void quick_sort_recursive(Student arr[], int low, int high, Comparator cmp) {
    if (low < high) {
        int mid = low + (high - low) / 2;

        if (cmp(&arr[mid], &arr[low]) < 0) swap_students(&arr[low], &arr[mid]);
        if (cmp(&arr[high], &arr[low]) < 0) swap_students(&arr[low], &arr[high]);
        if (cmp(&arr[high], &arr[mid]) < 0) swap_students(&arr[mid], &arr[high]);

        int pi = partition(arr, low, high, cmp);
        quick_sort_recursive(arr, low, pi - 1, cmp);
        quick_sort_recursive(arr, pi + 1, high, cmp);
    }
}

void quick_sort(Student arr[], int n, Comparator cmp) {
    quick_sort_recursive(arr, 0, n - 1, cmp);
}


// 4.3. AVL 트리 정렬 (AVL Tree Sort) - 균형 트리 적용

typedef struct node {
    Student data;
    struct node* left, * right;
    int height;
} Node;

Node* new_node(Student data) {
    Node* temp = (Node*)malloc(sizeof(Node));
    if (temp) {
        temp->data = data;
        moves++;
        temp->left = temp->right = NULL;
        temp->height = 1;
    }
    return temp;
}

int height(Node* N) { return (N == NULL) ? 0 : N->height; }
int get_balance(Node* N) { return (N == NULL) ? 0 : height(N->left) - height(N->right); }
int max(int a, int b) { return (a > b) ? a : b; }

Node* right_rotate(Node* y) {
    Node* x = y->left;
    Node* T2 = x->right;
    x->right = y;
    y->left = T2;
    y->height = max(height(y->left), height(y->right)) + 1;
    x->height = max(height(x->left), height(x->right)) + 1;
    return x;
}

Node* left_rotate(Node* x) {
    Node* y = x->right;
    Node* T2 = y->left;
    y->left = x;
    x->right = T2;
    x->height = max(height(x->left), height(x->right)) + 1;
    y->height = max(height(y->left), height(y->right)) + 1;
    return y;
}

// AVL 삽입 함수: 이제 ID가 아닌, 전달받은 cmp 함수를 사용합니다.
Node* avl_insert_node(Node* node, Student data, Comparator cmp) {
    if (node == NULL) return new_node(data);

    int cmp_result = cmp(&data, &node->data);
    comparisons++;

    if (cmp_result < 0) { // data가 node보다 작으면
        node->left = avl_insert_node(node->left, data, cmp);
    }
    else if (cmp_result > 0) { // data가 node보다 크면
        node->right = avl_insert_node(node->right, data, cmp);
    }
    else {
        // 중복 처리 (BST는 중복을 피해야 함. 여기서는 삽입을 무시)
        return node;
    }

    node->height = 1 + max(height(node->left), height(node->right));
    int balance = get_balance(node);

    // LL Case
    if (balance > 1 && cmp(&data, &node->left->data) < 0) return right_rotate(node);

    // RR Case
    if (balance < -1 && cmp(&data, &node->right->data) > 0) return left_rotate(node);

    // LR Case
    if (balance > 1 && cmp(&data, &node->left->data) > 0) {
        node->left = left_rotate(node->left);
        return right_rotate(node);
    }

    // RL Case
    if (balance < -1 && cmp(&data, &node->right->data) < 0) {
        node->right = right_rotate(node->right);
        return left_rotate(node);
    }

    return node;
}

void inorder_traversal(Node* root, Student arr[], int* index) {
    if (root != NULL) {
        inorder_traversal(root->left, arr, index);
        arr[(*index)++] = root->data;
        moves++;
        inorder_traversal(root->right, arr, index);
    }
}

void delete_tree(Node* node) {
    if (node == NULL) return;
    delete_tree(node->left);
    delete_tree(node->right);
    free(node);
}

void tree_sort_avl(Student arr[], int n, Comparator cmp) {
    Node* root = NULL;
    for (int i = 0; i < n; i++) {
        root = avl_insert_node(root, arr[i], cmp);
    }

    int index = 0;
    inorder_traversal(root, arr, &index);

    delete_tree(root);
}


// --- 5. 실행 및 측정 로직 ---

typedef void (*SortFunction)(Student[], int, Comparator);

void run_and_measure(const char* algo_name, SortFunction sort_func, Student original_data[], int n,
    const char* criterion_name, Comparator cmp) {

    // SKIP 로직 제거됨. 모든 조합 실행.

    long long total_comparisons = 0;
    long long total_moves = 0;
    clock_t start_time, end_time;
    double total_time = 0;

    for (int i = 0; i < REPETITIONS; i++) {
        Student* working_records = (Student*)malloc(n * sizeof(Student));
        if (!working_records) {
            perror("Memory allocation failed for working_records");
            return;
        }
        memcpy(working_records, original_data, sizeof(Student) * n);

        // 반복 시마다 무작위로 섞음 (첫 번째 실행 제외)
        if (i > 0) {
            shuffle_students(working_records, n);
        }

        comparisons = 0;
        moves = 0;

        start_time = clock();
        sort_func(working_records, n, cmp);
        end_time = clock();

        total_comparisons += comparisons;
        total_moves += moves;
        total_time += (double)(end_time - start_time) / CLOCKS_PER_SEC;

        free(working_records);
    }

    double avg_comparisons = (double)total_comparisons / REPETITIONS;
    double avg_moves = (double)total_moves / REPETITIONS;
    double avg_time_ms = (total_time / REPETITIONS) * 1000.0;

    printf("| %-18s | %-45s | %18.2f | %18.2f | %15.3fms |\n",
        algo_name, criterion_name, avg_comparisons, avg_moves, avg_time_ms);
}


// --- 6. 메인 함수 ---

int main() {
    // 난수 시드 초기화 (Shuffle 함수를 위해 필요)
    srand((unsigned int)time(NULL));

    int record_count = 0;
    Student* original_records = load_students(FILENAME, &record_count);

    if (record_count == 0) {
        printf("Error: No data loaded from %s.\n", FILENAME);
        return 1;
    }
    printf("Data loaded successfully. Total records: %d\n\n", record_count);

    typedef struct {
        const char* name;
        SortFunction func;
    } SortAlgo;

    typedef struct {
        const char* name;
        Comparator cmp;
        int is_stable; // Stability flag is no longer used for skipping, but kept for context
    } SortCriterion;

    SortAlgo algorithms[] = {
        {"Shell Sort (Ciura)", shell_sort},
        {"Quick Sort (Median)", quick_sort},
        {"AVL Tree Sort", tree_sort_avl}
    };

    SortCriterion criteria[] = {
        {"ID Ascending", compare_id_asc, 0},
        {"ID Descending", compare_id_desc, 0},
        {"NAME Ascending", compare_name_asc, 0},
        {"NAME Descending", compare_name_desc, 0},
        {"GENDER Ascending (Stable)", compare_gender_asc, 1},
        {"GENDER Descending (Stable)", compare_gender_desc, 1},
        {"Total Grade Ascending (Custom Tie)", compare_total_asc, 0},
        {"Total Grade Descending (Custom Tie)", compare_total_desc, 0}
    };

    printf("==============================================================================================================================\n");
    printf("| %-18s | %-45s | %-18s | %-18s | %-15s |\n",
        "Algorithm", "Criterion", "Avg Comparisons", "Avg Moves (Memory)", "Avg Time (ms)");
    printf("==============================================================================================================================\n");

    // 모든 조합 실행 및 측정 (SKIP 로직 제거)
    for (size_t i = 0; i < sizeof(algorithms) / sizeof(algorithms[0]); i++) {
        for (size_t j = 0; j < sizeof(criteria) / sizeof(criteria[0]); j++) {

            // 이전에 SKIP을 유발하던 모든 조건(is_stable, skip_for_duplicate)을 무시하고 실행합니다.

            run_and_measure(algorithms[i].name, algorithms[i].func,
                original_records, record_count,
                criteria[j].name, criteria[j].cmp);
        }
    }

    printf("==============================================================================================================================\n");

    free(original_records);

    return 0;
}
