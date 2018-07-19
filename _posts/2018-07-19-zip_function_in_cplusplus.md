---
layout: post
title: Как реализовать zip-function на языке C++
modified:
categories: 
excerpt: Building your own zip iterator adapter
tags: [C++]
image:
  feature: note.jpg
  thumb: thumb1.jpg
date: 2018-07-19T18:54:48+05:30
---
## Тема

 В этой заметке я покажу как написать аналог функции zip, известную из языков Python с помощью C++. Она будет работать только для векторов, содержащих значения типа double, чтобы не отвлекаться от механики итераторов. 

## Вступление

Я люблю Python за краткость и элегантность. Один из примеров проявления данной элегантности — реализация формул, например скалярного произведения. Если даны два математических вектора, то нахождение их скалярного произведения означает попарное умножение чисел на одинаковых позициях вектора, а затем суммирование этих умноженных значений. Скалярное произведение векторов (a, b, c) * (d, e, f) равно (a * d + b * e + c * f).

 Конечно, это можно сделать и с помощью языков C и C++. Код выглядел бы следующим образом:
{% highlight C++ %}
std::vector<double> a {1.0, 2.0, 3.0};
std::vector<double> b {4.0, 5.0, 6.0};
double sum {0};
for (size_t i {0}; i < a.size(); ++i) {
sum += a[i] * b[i];
}
// sum = 32.0
{% endhighlight %}
В Python такой код можно писать динамически одной строкой, не подключая конкретные функции библиотек:

    count = 0
    a = [1.0, 2.0, 3.0]
    b = [4.0, 5.0, 6.0]
    sum([p[0] * p[1] for p in zip(a,b)])
 
 Магическую функцию zip принимает два вектора a и b и преобразует их в смешанный вектор. Например, при вызове этой функции векторы [a1, a2, a3] и [b1, b2, b3] будут выглядеть как [ (a1, b1), (a2, b2), (a3, b3) ]. Это даёт возможность проитерировать по одному объединенному промежутку, выполнив попарное умножение и сложив результаты в переменную-аккумулятор. Приэтом не используются ни циклы, ни ненужные индексные переменные.

## Как это делается
// TODO
## Целый код

{% highlight C++ %}

#include <iostream>
#include <vector>
#include <numeric>

class zip_iterator
{
    using it_type = std::vector<double>::iterator;

    it_type it1;
    it_type it2;

public:
    zip_iterator(it_type iterator1, it_type iterator2)
        : it1{iterator1}, it2{iterator2}
    {}

    zip_iterator& operator++() {
        ++it1;
        ++it2;
        return *this;
    }

    bool operator!=(const zip_iterator& o) const {
        return it1 != o.it1 && it2 != o.it2;
    }

    bool operator==(const zip_iterator& o) const {
        return !operator!=(o);
    }

    std::pair<double, double> operator*() const {
        return {*it1, *it2};
    }
};

namespace std {

template <>
struct iterator_traits<zip_iterator> {
    using iterator_category = std::forward_iterator_tag;
    using value_type = std::pair<double, double>;
    using difference_type = long int;
};

}

class zipper {
    using vec_type = std::vector<double>;
    vec_type &vec1;
    vec_type &vec2;

public:
    zipper(vec_type &va, vec_type &vb)
        : vec1{va}, vec2{vb}
    {}

    zip_iterator begin() const { return {std::begin(vec1), std::begin(vec2)}; }
    zip_iterator end()   const { return {std::end(vec1),   std::end(vec2)}; }
};

using namespace std;

int main()
{
    vector<double> vec_a {1.0, 2.0, 3.0};
    vector<double> vec_b {4.0, 5.0, 6.0};

    zipper zipped {vec_a, vec_b};

    const auto add_product ([](double sum, const auto &p) {
        return sum + p.first * p.second;
    });

    const auto scalar_product (accumulate(begin(zipped), end(zipped), 0.0, add_product));

    cout << scalar_product << '\n';
}
 
{% endhighlight %}
