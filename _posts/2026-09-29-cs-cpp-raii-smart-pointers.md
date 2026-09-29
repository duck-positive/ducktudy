---
layout: post
title: "C++ RAII와 스마트 포인터 완전 정복: unique_ptr·shared_ptr·weak_ptr로 메모리 누수 없는 코드 작성하기"
date: 2026-09-29
categories: [cs, computer-science]
tags: [cpp, raii, smart-pointers, unique-ptr, shared-ptr, weak-ptr, memory-management, move-semantics, modern-cpp]
---

## RAII란 무엇인가

RAII(Resource Acquisition Is Initialization)는 C++의 핵심 관용구 중 하나로, **자원의 수명을 객체의 수명과 결합**시키는 프로그래밍 기법이다. Bjarne Stroustrup이 설계한 이 패턴의 핵심 원칙은 단순하다:

- **생성자**에서 자원을 획득한다
- **소멸자**에서 자원을 해제한다

여기서 "자원"은 힙 메모리만이 아니라 파일 핸들, 소켓, 뮤텍스 락, 데이터베이스 연결 등 명시적으로 해제해야 하는 모든 것을 포함한다. C++의 결정론적 소멸자(deterministic destructor) 호출 보장이 RAII의 토대가 된다. 스택에 놓인 객체는 스코프를 벗어날 때—예외가 발생하더라도—반드시 소멸자가 호출된다.

## 왜 RAII가 필요한가

원시 포인터(raw pointer)로 메모리를 직접 관리하면 여러 위험이 따른다.

**메모리 누수(Memory Leak)**: `new`로 할당한 뒤 `delete`를 빠뜨리거나, 예외 경로에서 `delete`가 실행되지 않을 때 발생한다.

**댕글링 포인터(Dangling Pointer)**: 이미 해제된 메모리를 가리키는 포인터를 역참조할 때 정의되지 않은 동작(Undefined Behavior)이 발생한다.

**이중 해제(Double Free)**: 같은 메모리를 두 번 `delete`하면 힙 자료구조가 손상된다.

```cpp
// 위험한 원시 포인터 사용 — 실제로 이렇게 쓰면 안 된다
#include <iostream>
#include <stdexcept>

void dangerous_function(bool throw_exception) {
    int* data = new int[100];  // 힙 할당

    if (throw_exception) {
        // delete[] data;  // 이 줄을 빠뜨리면 메모리 누수!
        throw std::runtime_error("예외 발생");
    }

    // 정상 경로에서도 delete를 잊기 쉽다
    delete[] data;
}

// RAII 방식 — 스코프를 벗어나면 자동 해제
#include <memory>
#include <vector>

void safe_function(bool throw_exception) {
    auto data = std::make_unique<int[]>(100);  // RAII 적용

    if (throw_exception) {
        throw std::runtime_error("예외 발생");
        // unique_ptr 소멸자가 자동으로 delete[] 호출
    }
    // 함수 종료 시 자동으로 해제 — 명시적 delete 불필요
}
```

## 스마트 포인터 3종 완전 정복

C++11부터 `<memory>` 헤더에 세 가지 스마트 포인터가 표준 라이브러리에 포함되었다.

### std::unique_ptr — 단독 소유권

`unique_ptr`는 자원에 대한 **배타적 소유권(exclusive ownership)**을 나타낸다. 복사(copy)는 불가능하고, 이동(move)만 가능하다. 소유권이 이전되면 원래 포인터는 `nullptr`가 된다. 오버헤드가 없으며, 원시 포인터와 동일한 성능을 낸다.

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <string>

// 파일 자원을 RAII로 관리하는 예시
class FileHandle {
public:
    explicit FileHandle(const std::string& path)
        : path_(path) {
        std::cout << "파일 열기: " << path_ << "\n";
        // 실제로는 fopen 등을 호출
    }

    ~FileHandle() {
        std::cout << "파일 닫기: " << path_ << "\n";
        // 실제로는 fclose 호출
    }

    void write(const std::string& content) {
        std::cout << "[" << path_ << "] 쓰기: " << content << "\n";
    }

    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

private:
    std::string path_;
};

// unique_ptr 기본 사용법
void demo_unique_ptr() {
    // make_unique 사용 권장 (new 직접 사용보다 예외 안전)
    auto file = std::make_unique<FileHandle>("output.txt");
    file->write("첫 번째 라인");
    file->write("두 번째 라인");
    // 스코프 종료 시 FileHandle 소멸자 자동 호출
}

// 소유권 이전 (move semantics)
std::unique_ptr<FileHandle> open_file(const std::string& path) {
    auto f = std::make_unique<FileHandle>(path);
    return f;  // 이동 시맨틱으로 반환 (복사 없음)
}

// unique_ptr의 컨테이너 사용
void demo_unique_ptr_vector() {
    std::vector<std::unique_ptr<FileHandle>> files;
    files.push_back(std::make_unique<FileHandle>("a.txt"));
    files.push_back(std::make_unique<FileHandle>("b.txt"));
    files.push_back(open_file("c.txt"));  // 소유권 이전

    for (auto& f : files) {
        f->write("내용");
    }
    // 벡터 소멸 시 모든 파일이 자동으로 닫힘
}

// 커스텀 Deleter
struct FreeDeleter {
    void operator()(void* p) const { free(p); }
};

void demo_custom_deleter() {
    // malloc으로 할당한 메모리를 unique_ptr로 관리
    std::unique_ptr<char, FreeDeleter> buf(
        static_cast<char*>(malloc(256))
    );
    snprintf(buf.get(), 256, "Hello, RAII!");
    std::cout << buf.get() << "\n";
    // 스코프 종료 시 FreeDeleter::operator()가 호출되어 free() 실행
}

int main() {
    demo_unique_ptr();
    demo_unique_ptr_vector();
    demo_custom_deleter();
    return 0;
}
```

### std::shared_ptr — 공유 소유권

`shared_ptr`는 **참조 카운팅(reference counting)**으로 자원의 공유 소유권을 구현한다. 여러 `shared_ptr`이 같은 자원을 가리킬 수 있으며, 마지막 `shared_ptr`가 소멸될 때 자원이 해제된다.

내부적으로 두 개의 포인터를 갖는다: 관리 객체를 가리키는 포인터와 제어 블록(control block, 참조 카운터와 weak 카운터 포함)을 가리키는 포인터. 이 때문에 `unique_ptr`보다 크기가 두 배이고, 참조 카운트 변경이 원자적(atomic) 연산이므로 약간의 성능 비용이 있다.

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <string>

class Connection {
public:
    explicit Connection(const std::string& url) : url_(url) {
        std::cout << "연결 생성: " << url_ << "\n";
    }
    ~Connection() {
        std::cout << "연결 해제: " << url_ << "\n";
    }
    void query(const std::string& sql) {
        std::cout << "[" << url_ << "] 쿼리: " << sql << "\n";
    }
private:
    std::string url_;
};

// 커넥션 풀 시뮬레이션: 여러 서비스가 커넥션을 공유
class ConnectionPool {
public:
    std::shared_ptr<Connection> get() {
        if (!conn_) {
            conn_ = std::make_shared<Connection>("db://localhost:5432");
        }
        return conn_;  // 소유권 공유
    }
private:
    std::shared_ptr<Connection> conn_;
};

class UserService {
public:
    explicit UserService(std::shared_ptr<Connection> conn)
        : conn_(std::move(conn)) {}

    void find_user(int id) {
        conn_->query("SELECT * FROM users WHERE id = " + std::to_string(id));
    }
private:
    std::shared_ptr<Connection> conn_;
};

class OrderService {
public:
    explicit OrderService(std::shared_ptr<Connection> conn)
        : conn_(std::move(conn)) {}

    void find_orders(int user_id) {
        conn_->query("SELECT * FROM orders WHERE user_id = " + std::to_string(user_id));
    }
private:
    std::shared_ptr<Connection> conn_;
};

void demo_shared_ptr() {
    ConnectionPool pool;
    auto conn = pool.get();

    // 두 서비스가 같은 커넥션을 공유
    UserService  user_svc(conn);
    OrderService order_svc(conn);

    std::cout << "use_count: " << conn.use_count() << "\n";  // 3

    user_svc.find_user(42);
    order_svc.find_orders(42);

    // conn 변수를 초기화해도 아직 서비스들이 참조 중
    conn.reset();
    std::cout << "use_count after reset: " << conn.use_count() << "\n";  // 0 (conn은 null)
    // 두 서비스가 소멸되어야 실제 Connection이 해제됨
}

int main() {
    demo_shared_ptr();
    return 0;
}
```

### std::weak_ptr — 순환 참조 해결

`shared_ptr`만 사용할 때 발생하는 **순환 참조(circular reference)** 문제를 해결하기 위해 `weak_ptr`가 존재한다. `weak_ptr`는 자원을 소유하지 않으며(참조 카운트를 증가시키지 않음), 자원이 유효한지 확인한 후 `shared_ptr`로 승격해야 사용할 수 있다.

```cpp
#include <iostream>
#include <memory>
#include <string>

// 순환 참조 문제 발생 예시 (절대 이렇게 하면 안 됨)
struct BadNode {
    std::string name;
    std::shared_ptr<BadNode> next;  // 순환 시 메모리 누수!
    ~BadNode() { std::cout << name << " 소멸\n"; }
};

// weak_ptr로 해결
struct Node {
    std::string name;
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // weak_ptr로 역방향 링크
    ~Node() { std::cout << name << " 소멸\n"; }
};

void demo_weak_ptr() {
    auto n1 = std::make_shared<Node>();
    auto n2 = std::make_shared<Node>();
    n1->name = "Node1";
    n2->name = "Node2";

    n1->next = n2;      // n1 → n2 (shared, 강한 참조)
    n2->prev = n1;      // n2 → n1 (weak, 약한 참조 — 순환 참조 방지)

    // weak_ptr 사용: 유효성 확인 후 승격
    if (auto locked = n2->prev.lock()) {
        std::cout << "이전 노드: " << locked->name << "\n";
    }

    // Observer 패턴에서 weak_ptr 활용
    struct Subject;
    struct Observer {
        std::string name;
        void update(const std::string& msg) {
            std::cout << name << " 수신: " << msg << "\n";
        }
    };

    std::vector<std::weak_ptr<Observer>> observers;
    {
        auto obs1 = std::make_shared<Observer>();
        obs1->name = "Observer1";
        observers.push_back(obs1);

        auto obs2 = std::make_shared<Observer>();
        obs2->name = "Observer2";
        observers.push_back(obs2);

        // obs1, obs2가 스코프 종료 후 소멸
    }

    // 소멸된 Observer는 자동 제거
    std::cout << "유효한 Observer:\n";
    for (auto& w : observers) {
        if (auto obs = w.lock()) {  // 유효성 확인 필수
            obs->update("이벤트");
        } else {
            std::cout << "(소멸된 옵저버 건너뜀)\n";
        }
    }
}

int main() {
    demo_weak_ptr();
    return 0;
}
```

## 이동 시맨틱(Move Semantics)과 스마트 포인터

C++11의 이동 시맨틱은 RAII와 스마트 포인터를 더욱 강력하게 만든다. `unique_ptr`의 소유권 이전은 이동 시맨틱으로 구현되며, 불필요한 복사 없이 자원을 전달한다.

```cpp
#include <memory>
#include <vector>

class Widget {
public:
    explicit Widget(int id) : id_(id) {}
    int id() const { return id_; }
private:
    int id_;
};

// 팩토리 함수: 이동 시맨틱으로 효율적 반환
std::unique_ptr<Widget> make_widget(int id) {
    return std::make_unique<Widget>(id);  // NRVO 또는 이동
}

// 소유권을 함수에 이전
void process(std::unique_ptr<Widget> w) {
    // 이 함수가 Widget의 소유자
    std::cout << "처리 중: Widget " << w->id() << "\n";
}  // 함수 종료 시 Widget 소멸

// 소유권 공유 (non-owning 참조)
void inspect(const Widget& w) {
    std::cout << "검사 중: Widget " << w.id() << "\n";
}

int main() {
    auto w = make_widget(42);
    inspect(*w);            // 소유권 유지, 참조만 전달
    process(std::move(w)); // 소유권 이전
    // w는 이제 nullptr

    // 벡터에서 이동
    std::vector<std::unique_ptr<Widget>> widgets;
    for (int i = 0; i < 5; ++i) {
        widgets.push_back(make_widget(i));
    }
    return 0;
}
```

## 주의사항과 실전 팁

**make_unique/make_shared를 사용하라**: `new`를 직접 사용하면 예외 안전성 문제가 생길 수 있다. `f(std::shared_ptr<T>(new T), g())`에서 `new T`와 `g()` 호출 순서가 보장되지 않아 `g()`가 예외를 던지면 메모리가 누수될 수 있다.

**소유권 정책을 명확히 하라**: 기본값은 `unique_ptr`다. 공유가 정말 필요할 때만 `shared_ptr`를 쓴다. 참조만 필요하고 소유하지 않는다면 원시 포인터(non-owning pointer)나 참조를 사용한다.

**순환 참조를 주의하라**: `shared_ptr`로 그래프 구조를 만들면 사이클이 발생하기 쉽다. 부모→자식은 `shared_ptr`, 자식→부모는 `weak_ptr`로 설계하라.

**`this`로 shared_ptr 만들기**: 멤버 함수 내에서 `shared_ptr<T>(this)`를 만들면 두 개의 독립된 제어 블록이 생겨 이중 해제가 발생한다. 이 경우 `std::enable_shared_from_this<T>`를 상속하고 `shared_from_this()`를 사용한다.

**성능 고려**: `shared_ptr`의 atomic 참조 카운트는 멀티스레드 환경에서도 안전하지만 비용이 있다. 불필요한 `shared_ptr` 복사를 피하고, 함수 인자로는 참조로 받는다 (`const shared_ptr<T>&`).

## 참고 자료
- [cppreference.com - std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)
- [cppreference.com - std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)
- [isocpp.org C++ Core Guidelines - Resource Management](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-resource)
- [cppreference.com - std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)
