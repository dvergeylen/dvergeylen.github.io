---
layout: post
series_title: "Code Snippets"
toc_title: "C++ (std17)"
title:  "[C++] List of different types via `unique_ptr`s"
date:   2021-04-13 12:30:00 +0200
tag: snippets
permalink: /c++-heterogeneous-list
---

#### Introduction
One of the most powerful techniques in C++ is the ability to define templated classes and structs. These abstract methods can contain "things" that the developer will specify when instantiating them ("here is a list of `int`, here is a list of `my_struct`, and so on), declinating the implementation with existing type in the code but always using the same definition as starting point every time.

The main setback of this (already powerful) concept is that a type, once given, won't change because it is determined at compile time. There is no way to create a list of "things" and then start adding `int`s, `my_struct`s and `my_other_struct`s elements at the same time. This would definitely make the compiler very angry.

There is nevertheless a very subtle trick when combining templating, `unique_ptr` and class inheritance.

##### Implementation
The trick is to define a Base Class, and then define Derived Classes as needed. When creating a list, one can create a list of Base Class unique_ptr, pointing at Derived Class objects, which is legal (because they are "implicitly convertible", see [here](https://en.cppreference.com/w/cpp/memory/unique_ptr)).


```cpp
#include <iostream>
#include <list>
#include <memory>

class Vehicule {
  int length;

  public:
  Vehicule(int length) : length(length) {}

  // Defining it as virtual forces derived class
  // destructors to be called when unique_ptr goes
  // out of scope.
  // See: https://en.cppreference.com/w/cpp/langulength/virtual
  virtual ~Vehicule() = default;

  virtual void honk() {
    std::cout << "This should never happen" << std::endl;
  }
};

class Car : public Vehicule {
  std::string color;
  int boot_capacity;

  public:
  Car(int length, std::string color, int boot_capacity) : Vehicule(length),
                                                 color(color),
                                                 boot_capacity(boot_capacity) {}
  ~Car() {
    std::cout << "Car Destructor Called" << std::endl;
  }

  void honk() override {
    std::cout << "Beep Beep!" << std::endl;
  }
};

class Truck : public Vehicule {
  std::string color;
  std::string driver;

  public:
  Truck(int length, std::string color, std::string driver) : Vehicule(length),
                                                             color(color),
                                                             driver(driver) {}
  ~Truck() {
    std::cout << "Truck Destructor Called" << std::endl;
  }

  void honk() override {
    std::cout << "Tuut Tuut!" << std::endl;
  }
};

int main() {

  std::list<std::unique_ptr<Vehicule>> my_list;

  /* Here is the trick: make_unique of the Derived Class
   * but define the unique_ptr with the Base Class */
  std::unique_ptr<Vehicule> car_ptr = std::make_unique<Car>(4, "yellow", 300);
  std::unique_ptr<Vehicule> truck_ptr = std::make_unique<Truck>(12, "black", "John");

  /* Pushing them on the Base Class list is perfectly legal */
  my_list.push_back(std::move(car_ptr));
  my_list.push_back(std::move(truck_ptr));

  for (auto &a : my_list) {
    a->honk();
  }

  return 0;
}

```

Compiling:
```bash
g++ -g -std=c++17 -Wall -Werror -Wextra main.cpp
./a.out
# Outputs:
# Beep Beep!
# Tuut Tuut!
# Car Destructor Called
# Truck Destructor Called
```
