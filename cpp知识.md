
animal 是基，dog是派生。
多态：基类指针可指向派生对象


好的！我们先解释一些基本概念，然后通过一个例子来讲解 `virtual` 析构函数的作用。

---

### **基本概念**

1. **基类（Base Class）和派生类（Derived Class）：**
   • **基类**：是一个通用的类，定义了某些通用的属性和行为。
   • **派生类**：是从基类继承而来的类，继承了基类的属性和行为，同时可以扩展或修改这些属性和行为。
   • 例如，`Animal` 是一个基类，`Dog` 和 `Cat` 是从 `Animal` 派生出来的类。

2. **基类指针：**
   • 基类指针是指向基类对象的指针，但它也可以指向派生类对象。
   • 这种特性是实现**多态**的基础。
   • 例如，`Animal*` 是一个基类指针，它可以指向 `Animal` 对象，也可以指向 `Dog` 或 `Cat` 对象。

3. **虚函数（Virtual Function）：**
   • 虚函数是在基类中声明为 `virtual` 的函数，派生类可以重写（override）它。
   • 通过基类指针调用虚函数时，实际调用的是派生类的实现（如果派生类重写了该函数）。
   • 虚函数是实现**运行时多态**的关键。

4. **虚析构函数（Virtual Destructor）：**
   • 析构函数用于释放对象占用的资源。
   • 如果基类的析构函数是虚函数（`virtual`），那么当通过基类指针删除派生类对象时，会先调用派生类的析构函数，再调用基类的析构函数。
   • 如果基类的析构函数不是虚函数，则只会调用基类的析构函数，可能导致派生类的资源未被正确释放。

---

### **例子讲解**

#### 场景描述：
假设我们有一个基类 `Animal` 和一个派生类 `Dog`。`Animal` 类有一个析构函数，`Dog` 类也有一个析构函数。我们通过基类指针 `Animal*` 来管理 `Dog` 对象，并观察析构函数的调用情况。

#### 代码实现：

```cpp
#include <iostream>

// 基类
class Animal {
public:
    Animal() {
        std::cout << "Animal constructor called." << std::endl;
    }

    // 虚析构函数
    virtual ~Animal() {
        std::cout << "Animal destructor called." << std::endl;
    }
};

// 派生类
class Dog : public Animal {
public:
    Dog() {
        std::cout << "Dog constructor called." << std::endl;
    }

    ~Dog() {
        std::cout << "Dog destructor called." << std::endl;
    }
};

int main() {
    // 基类指针指向派生类对象
    Animal* myAnimal = new Dog();

    // 删除对象
    delete myAnimal;

    return 0;
}
```

---

#### **运行结果：**

```
Animal constructor called.
Dog constructor called.
Dog destructor called.
Animal destructor called.
```

---

#### **分析：**

1. **构造顺序：**
   • 先调用基类 `Animal` 的构造函数，再调用派生类 `Dog` 的构造函数。

2. **析构顺序：**
   • 由于 `Animal` 的析构函数是虚函数，当通过基类指针 `Animal*` 删除 `Dog` 对象时：
     ◦ 先调用派生类 `Dog` 的析构函数。
     ◦ 再调用基类 `Animal` 的析构函数。
   • 如果 `Animal` 的析构函数不是虚函数，则只会调用 `Animal` 的析构函数，`Dog` 的析构函数不会被调用，可能导致资源泄漏。

---

### **如果没有虚析构函数会发生什么？**

如果将 `Animal` 的析构函数改为非虚函数：

```cpp
class Animal {
public:
    Animal() {
        std::cout << "Animal constructor called." << std::endl;
    }

    // 非虚析构函数
    ~Animal() {
        std::cout << "Animal destructor called." << std::endl;
    }
};
```

运行结果：

```
Animal constructor called.
Dog constructor called.
Animal destructor called.
```

可以看到，`Dog` 的析构函数没有被调用！如果 `Dog` 类中有动态分配的资源（如内存、文件句柄等），这些资源将不会被正确释放，导致内存泄漏或其他问题。

---

### **总结**

• **虚析构函数的作用：**  
  确保通过基类指针删除派生类对象时，派生类的析构函数被正确调用，从而避免资源泄漏。

• **使用场景：**  
  如果一个类可能被继承，并且可能通过基类指针删除派生类对象，那么基类的析构函数应该声明为虚函数。

• **注意事项：**  
  虚析构函数会引入额外的开销（虚函数表），因此不要滥用，只在必要时使用。

希望这个例子能帮助你理解虚析构函数的作用！如果还有疑问，欢迎继续提问！ 😊