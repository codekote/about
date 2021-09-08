# ООП в Си. Разные подходы в реализации.
Собрал из разных источников заметку о различных подходах к реализации ООП в Си.

## #1. 
@kafeman описал в своём [комметарии на Хабре](https://habr.com/ru/post/263547/comments/#comment_8515227). Для меня не совсем ясен постфикс `Private`, все структуры находятся в глобальной области видимости, методы тоже. Достучаться до члена `x` “приватной” структуры `Point2DPrivate` можно через `point->parent.private->x`. Возможно автор хотел показать общий подход к построению классов/объектов и не собирался описывать сокрытие.


``` c
// main.c
#include <stdio.h>
#include <stdlib.h>

typedef struct _Point2DPrivate {
    int x;
    int y;
} Point2DPrivate;

typedef struct _Point2D {
    Point2DPrivate *private;
} Point2D;

void Point2D_Constructor(Point2D *point) {
    point->private = malloc(sizeof(Point2DPrivate));
    point->private->x = 0;
    point->private->y = 0;
}

void Point2D_Destructor(Point2D *point) {
    free(point->private);
}

void Point2D_SetX(Point2D *point, int x) {
    point->private->x = x;
}

int Point2D_GetX(Point2D *point) {
    return point->private->x;
}

void Point2D_SetY(Point2D *point, int y) {
    point->private->y = y;
}

int Point2D_GetY(Point2D *point) {
    return point->private->y;
}

Point2D *Point2D_New(void) {
    Point2D *point = malloc(sizeof(Point2D));
    Point2D_Constructor(point); /* <-- Вызываем конструктор */
    return point;
}

void Point2D_Delete(Point2D *point) {
    Point2D_Destructor(point); /* <-- Вызываем деструктор */
    free(point);
}

typedef struct _Point3DPrivate {
    int z;
} Point3DPrivate;

typedef struct _Point3D {
    Point2D parent;
    Point3DPrivate *private;
} Point3D;

void Point3D_Constructor(Point3D *point) {
    Point2D_Constructor(&point->parent); /* <-- Вызываем родительский конструктор! */
    point->private = malloc(sizeof(Point3DPrivate));
    point->private->z = 0;
}

void Point3D_Destructor(Point3D *point) {
    Point2D_Destructor(&point->parent); /* <-- Вызываем родительский деструктор! */
    free(point->private);
}

void Point3D_SetZ(Point3D *point, int z) {
    point->private->z = z;
}

int Point3D_GetZ(Point3D *point) {
    return point->private->z;
}

Point3D *Point3D_New(void) {
    Point3D *point = malloc(sizeof(Point3D));
    Point3D_Constructor(point); /* <-- Вызываем конструктор */
    return point;
}

void Point3D_Delete(Point3D *point) {
    Point3D_Destructor(point); /* <-- Вызываем деструктор */
    free(point);
}

int main(int argc, char **argv) {
    /* Создаем экземпляр класса Point3D */
    Point3D *point = Point3D_New();
    
    /* Устанавливаем x и y координаты */
    Point2D_SetX((Point2D*)point, 10);
    Point2D_SetY((Point2D*)point, 15);
    
    /* Теперь z координата */
    Point3D_SetZ(point, 20);
    
    /* Должно вывести: x = 10, y = 15, z = 20 */
    printf("x = %d, y = %d, z = %d\n",
           Point2D_GetX((Point2D*)point),
           Point2D_GetY((Point2D*)point),
           Point3D_GetZ(point));
    
    /* Удаляем объект */
    Point3D_Delete(point);
    
    return 0;
}
```

Сам же [пост](https://habr.com/ru/post/263547/), на который @kafeman оставил комментарий, описывает подход сокрытия через возможности спецификатора `static`.

Цитата @dehasi:
> Имея такую возможность, можно разделить public и private данные по разным файлам, а в структуре хранить только указатель на приватные данные. Понадобится две структуры: одна приватная, а вторая с методами для работы и указателем на приватную. Чтобы вызывать функции на объекте, договоримся первым параметром передавать указатель на структуру, которая ее вызывает. Объявим структуру с сеттерами, геттерами и указателем на приватное поле, а также функции, которые будут создавать структуру и удалять.

``` c
//point2d.h
typedef struct point2D {
  void *prvtPoint2D;
  int (*getX) (struct point2D*);
  void (*setX)(struct point2D*, int);
 //...
} point2D;
point2D* newPoint2D();
void deletePoint2D(point2D*);
```
Цитата @dehasi:
> Здесь будет инициализироваться приватное поле и указатели на функции, чтобы с этой структурой можно было работать.

``` c
//point2d.c
#include <stdlib.h>
#include "point2d.h"
typedef struct private {
  int x;
  int y;
} private;

static int getx(struct point2D*p) {
   return ((struct private*)(p->prvtPoint2D))->x;
}
static void setx(struct point2D *p, int val) {
    ((struct private*)(p->prvtPoint2D))->x = val;
}

point2D* newPoint2D()  {
  point2D* ptr;
  ptr = (point2D*) malloc(sizeof(point2D));
  ptr -> prvtPoint2D = malloc(sizeof(private));
  ptr -> getX = &getx;
  ptr -> setX = &setx;
  // ....
  return ptr;
}
```
Цитата @dehasi:
> Теперь, работа с этой структурой, может осуществляться с помощью сеттеров и геттеров.

``` c
// main.c
#include <stdio.h>
#include "point2d.h"

int main() {
  point2D *point = newPoint2D();
  int p = point->getX(point);
  point->setX(point, 42);
  p = point->getX(point);
  printf("p = %d\n", p);
  deletePoint2D(point);
  return 0;
}
```

## #2. 
В книге Extreme C автор Amini K. постарался описать всё: инкапсуляцию и сокрытие, типы связей объектов/классов (наследование, агрегация, композиция), а также полиморфизм. И всё это на чистом Си. Автор создал [репозиторий](https://github.com/PacktPublishing/Extreme-C), где опубликовал весь исходный код, который был в книге. По разделам можно смотреть здесь:

* [Chapter 6: OOP and Encapsulation](https://github.com/PacktPublishing/Extreme-C/tree/master/ch06-oop-and-encapsulation)
* [Chapter 7: Composition and Aggregation](https://github.com/PacktPublishing/Extreme-C/tree/master/ch07-composition-and-aggregation)
* [Chapter 8: Inheritance and Polymorphism](https://github.com/PacktPublishing/Extreme-C/tree/master/ch07-composition-and-aggregation)

Основная идея заключается в разделении облести видимости по файлам. Каждый родительский объект имеет интерфейс только к дочернему. Ниже отрывки из кода (не полностью).
``` c
// class_and_methods.c — описываются данные и методы

typedef int bool_t;

// Декларация структуры
typedef struct {
  size_t size;
  int* items;
} list_t;

//...
// Приватный метод с префиксом __
bool_t __list_is_full(list_t* list) {
  return (list->size == MAX_SIZE);
}

// Публичный метод
size_t list_size(list_t* list) {
  return list->size;
}
//...
```

``` c
// public.h - заголовочный файл раскрывает вышестоящему только те данные и методы, которые в нём описаны. В данном случае main.c имеет доступ только к публичным методам.
...
struct list_t; // предватительная декларация

// Публичные методы
int list_add(struct list_t*, int);
size_t list_size(struct list_t*);
//...
```

``` c
// main.c
#include <public.h>
int main(int argc, char** argv) {
    //...
    list_add(list1, 4);
    list_size(source);
    //...
}
```
Сохраню здесь, буду периодически подглядывать 👀.
