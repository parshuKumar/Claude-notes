# STEP 0 — Diagnostic (15 questions)

All C++ snippets verified to compile and run under
**Apple clang 21.0.0, `-std=c++20 -Wall -Wextra`, arm64-apple-darwin25.3.0.**
Every snippet is self-contained and complete.

Answer with exact stdout, in order. If you think it does not compile,
say so and name the reason. If you think it is UB, say so and give the
outcome you'd expect on this compiler.

---

## Part 1 — Predict the output (Q1–Q12)

### Q1
```cpp
#include <iostream>
struct Base {
    Base()          { std::cout << "Base ctor: "; log(); }
    virtual ~Base() { std::cout << "Base dtor: "; log(); }
    virtual void log() const { std::cout << "Base\n"; }
};
struct Derived : Base {
    void log() const override { std::cout << "Derived\n"; }
};
int main() { Derived d; }
```

### Q2
```cpp
#include <iostream>
#include <vector>
struct Shape { virtual void draw() const { std::cout << "Shape\n"; } virtual ~Shape() = default; };
struct Circle : Shape { void draw() const override { std::cout << "Circle\n"; } };
void render(Shape s)           { s.draw(); }
void renderRef(const Shape& s) { s.draw(); }
int main() {
    Circle c;
    render(c);
    renderRef(c);
    std::vector<Shape> v;
    v.push_back(Circle{});
    v[0].draw();
}
```

### Q3
```cpp
#include <iostream>
struct Account {
    int fee;
    int balance;
    Account(int b) : balance(b), fee(balance / 10) {
        std::cout << "balance=" << balance << " fee=" << fee << "\n";
    }
};
int main() { Account a(500); }
```

### Q4
```cpp
#include <iostream>
struct Base {
    void speak()         { std::cout << "Base()\n"; }
    void speak(int n)    { std::cout << "Base(int " << n << ")\n"; }
    void speak(double d) { std::cout << "Base(double " << d << ")\n"; }
};
struct Derived : Base {
    void speak(const char* s) { std::cout << "Derived(str " << s << ")\n"; }
};
int main() {
    Derived d;
    d.speak("hi");
    d.speak(42);
}
```

### Q5
```cpp
#include <iostream>
struct Base           { virtual void pay(int amount = 100) { std::cout << "Base " << amount << "\n"; }
                        virtual ~Base() = default; };
struct Derived : Base { void pay(int amount = 500) override { std::cout << "Derived " << amount << "\n"; } };
int main() {
    Base* p = new Derived();
    p->pay();
    Derived d;
    d.pay();
    delete p;
}
```

### Q6
```cpp
#include <iostream>
struct Logger { ~Logger() { std::cout << "~Logger\n"; } };
struct Base   { ~Base()   { std::cout << "~Base\n"; } };
struct Derived : Base {
    Logger lg;
    ~Derived() { std::cout << "~Derived\n"; }
};
int main() {
    Base* p = new Derived();
    delete p;
    std::cout << "---\n";
    Derived d;
}
```

### Q7
```cpp
#include <iostream>
struct Person                    { Person()   { std::cout << "Person\n"; }   virtual ~Person() = default; };
struct Employee : virtual Person { Employee() { std::cout << "Employee\n"; } };
struct Student  : virtual Person { Student()  { std::cout << "Student\n"; } };
struct Intern : Employee, Student { Intern()  { std::cout << "Intern\n"; } };
int main() { Intern i; }
```

### Q8
```cpp
#include <iostream>
struct Animal {
    virtual void speak() const { std::cout << "Animal\n"; }
    virtual ~Animal() = default;
};
struct Dog : Animal {
    void speak() { std::cout << "Dog\n"; }
};
int main() {
    Dog d;
    Animal* a = &d;
    a->speak();
    d.speak();
}
```

### Q9
```cpp
#include <iostream>
#include <stdexcept>
struct Member { Member()  { std::cout << "Member ctor\n"; }
                ~Member() { std::cout << "Member dtor\n"; } };
struct Base   { Base()    { std::cout << "Base ctor\n"; }
                ~Base()   { std::cout << "Base dtor\n"; } };
struct Derived : Base {
    Member m;
    Derived()  { std::cout << "Derived body\n"; throw std::runtime_error("boom"); }
    ~Derived() { std::cout << "Derived dtor\n"; }
};
int main() {
    try { Derived d; }
    catch (const std::exception& e) { std::cout << "caught " << e.what() << "\n"; }
}
```

### Q10
```cpp
#include <iostream>
struct Tracer { Tracer(const char* n) : name(n) { std::cout << "ctor " << name << "\n"; }
                const char* name; };
Tracer global("global");
Tracer& meyers() { static Tracer t("meyers"); return t; }
int main() {
    std::cout << "main start\n";
    meyers();
    meyers();
    std::cout << "main end\n";
}
```

### Q11
```cpp
#include <iostream>
class Report {
public:
    void generate() { header(); body(); }
    virtual ~Report() = default;
private:
    virtual void header() { std::cout << "generic header\n"; }
    virtual void body()   { std::cout << "generic body\n"; }
};
class SalesReport : public Report {
private:
    void header() override { std::cout << "sales header\n"; }
};
int main() {
    Report* r = new SalesReport();
    r->generate();
    delete r;
}
```

### Q12
```cpp
#include <iostream>
struct Buffer {
    int* data;
    Buffer(int v) : data(new int(v)) {}
    ~Buffer() { delete data; }
    void set(int v) { *data = v; }
    int get() const { return *data; }
};
int main() {
    Buffer a(1);
    Buffer b = a;
    b.set(99);
    std::cout << a.get() << " " << b.get() << std::endl;
    std::cout << "end of main" << std::endl;
}
```

---

## Part 2 — Design judgment (Q13–Q15)

Two to four sentences each. What is wrong, and what would you do instead.

### Q13
```cpp
class Rectangle {
public:
    virtual void setWidth(int w)  { width_  = w; }
    virtual void setHeight(int h) { height_ = h; }
    int area() const { return width_ * height_; }
protected:
    int width_ = 0, height_ = 0;
};

class Square : public Rectangle {
public:
    void setWidth(int w)  override { width_ = height_ = w; }
    void setHeight(int h) override { width_ = height_ = h; }
};

// elsewhere, written against Rectangle:
void resize(Rectangle& r) {
    r.setWidth(5);
    r.setHeight(4);
    assert(r.area() == 20);
}
```

### Q14
```cpp
class Employee { /* name, id, base pay */ };

class FullTimeEmployee        : public Employee {};
class PartTimeEmployee        : public Employee {};
class ContractEmployee        : public Employee {};

class FullTimeManager         : public FullTimeEmployee {};
class PartTimeManager         : public PartTimeEmployee {};
class ContractManager         : public ContractEmployee {};

class FullTimeRemoteManager   : public FullTimeManager {};
class PartTimeRemoteManager   : public PartTimeManager {};
// ... and payroll needs to add "intern" and "seasonal" next sprint
```

### Q15
```cpp
class PaymentProcessor {
public:
    virtual void charge(double amount)          = 0;
    virtual void refund(double amount)          = 0;
    virtual void authorize(double amount)       = 0;
    virtual void capture(double amount)         = 0;
    virtual void startSubscription(int planId)  = 0;
    virtual void cancelSubscription(int planId) = 0;
    virtual void issueGiftCard(double amount)   = 0;
    virtual ~PaymentProcessor() = default;
};

class CashOnDelivery : public PaymentProcessor {
public:
    void charge(double amount) override { /* real */ }
    void refund(double) override          { throw std::logic_error("not supported"); }
    void authorize(double) override       { throw std::logic_error("not supported"); }
    void capture(double) override         { throw std::logic_error("not supported"); }
    void startSubscription(int) override  { throw std::logic_error("not supported"); }
    void cancelSubscription(int) override { throw std::logic_error("not supported"); }
    void issueGiftCard(double) override   { throw std::logic_error("not supported"); }
};
```
