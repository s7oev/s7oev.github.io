---
layout: post
title: "Java-like Object.equals() behavior in ABAP"
date: 2020-05-15
---

Java’s equals() can be very useful: using it allows us to compare objects by their attributes. This is opposed to the traditional comparison with ‘==’ that compares whether two object references point to the same memory address.

For example, if we define a Car class with the attributes producer, model and year, and then we define two new objects of class Car – car1 and car2 – and we set each to be a Mercedes (producer) E400 (model) from 2016 (year), comparing them with ‘==’ would return false, while doing car1.equals(car2) would return true.

Today, I was wondering how we can achieve this in ABAP; unfortunately, it turns out, SAP does not provide a default way to do this. However, a coworker managed to find the if_xlf_comparable interface that requires you to implement an equals() method. While this specific interface is supposed to be used only with XLF-related objects, as evident by the name, its signature seemed general enough, so I used it as a basis, and for the actual implementation, I followed the Java way.

<h2>Defining a class to implement equals()</h2>
To continue with the example described above, I defined the Car class below, with its relevant attributes:

```
CLASS car DEFINITION.
  PUBLIC SECTION.
    METHODS:
      constructor IMPORTING producer TYPE string
                            model    TYPE string
                            year     TYPE i.

  PRIVATE SECTION.
    DATA:
      producer TYPE string,
      model    TYPE string,
      year     TYPE i.
ENDCLASS.
```

Then, in the public methods, I defined the equals method by copy-pasting the signature from the above-mentioned interface:

```
METHODS:
  equals IMPORTING object          TYPE REF TO object
         RETURNING VALUE(is_equal) TYPE abap_bool.
```

<h2>Writing the equals() method</h2>
Then, obviously, the most important task remained: writing the equals() method. The traditional way to implement equals() in Java is as follows:

1. Check if the other object reference is actually a reference to the same object; if so, return true
2. Check if the other object reference is null; if so, return false
3. Check if the other object’s class is the same as the caller object’s class; if not so, return false
4. Downcast the other object to the relevant class and compare the relevant attributes; if all relevant attributes are equal, return true

An excellent example for this is Pricenton’s implementation in their class representing a point in a 2-dimensional space – Point2D (<a href="https://algs4.cs.princeton.edu/13stacks/Point2D.java.html">source</a>):

```
public boolean equals(Object other) {
    if (other == this) return true;
    if (other == null) return false;
    if (other.getClass() != this.getClass()) return false;
    Point2D that = (Point2D) other;
    return this.x == that.x && this.y == that.y;
}
```

Then, I implemented the code above in ABAP, with the obvious difference that instead of comparing a point’s x and y coordinate, I was to compare the producer, model, and year attributes in my Car class.

```
METHOD equals.
  IF object EQ me.
    is_equal = abap_true.
    RETURN.
  ENDIF.

  IF object IS NOT BOUND.
    is_equal = abap_false.
    RETURN.
  ENDIF.

  TRY.
      DATA(other) = CAST car( object ).
      is_equal = xsdbool( me->producer EQ other->producer AND me->model EQ other->model AND me->year EQ other->year ).
    CATCH cx_sy_move_cast_error.
      is_equal = abap_false.
  ENDTRY.
ENDMETHOD.
```

<h2>Simple demo</h2>
To test this, I created a few objects of the class, as well as one from a Truck class (refer to the full code at the end for its definition), and used the equals method in all possible cases:

```
DATA(car1) = NEW car( producer = 'Mercedes' model = 'E400' year = 2016 ).
DATA(car2) = NEW car( producer = 'BMW' model = 'X5' year = 2013 ).
DATA(car3) = NEW car( producer = 'Mercedes' model = 'E400' year = 2016 ).
DATA car4 TYPE REF TO car.
DATA(car5) = car1.
DATA(truck1) = NEW truck( producer = 'CAT' model = 'CT660' ).

" 2 different non-null Car objects with different attributes
cl_demo_output=>write( |Car 1 and Car 2 are the same:   { car1->equals( car2 ) }| ).

" 2 different non-null Car objects with the same attributes
cl_demo_output=>write( |Car 1 and Car 3 are the same:   { car1->equals( car3 ) }| ).

" 1 non-null Car object with 1 null Car object
cl_demo_output=>write( |Car 1 is the same as Car 4:     { car1->equals( car4 ) }| ).

" 2 different references to the same Car object
cl_demo_output=>write( |Car 1 is the same as Car 5:     { car1->equals( car5 ) }| ).

" Self as an argument
cl_demo_output=>write( |Car 1 is the same as itself:    { car1->equals( car1 ) }| ).

" 1 Car object with 1 Truck object
cl_demo_output=>write( |Car 1 and Truck 1 are the same: { car1->equals( truck1 ) }| ).

cl_demo_output=>display(  ).
```

After running this code, we have the following output:

```
Car 1 and Car 2 are the same:   
Car 1 and Car 3 are the same:   X
Car 1 is the same as Car 4:     
Car 1 is the same as Car 5:     X
Car 1 is the same as itself:    X
Car 1 and Truck 1 are the same: 
```

<h2>Closing notes</h2>
To conclude, this article demonstrates how to implement an equals() method in ABAP the same way that you would do it in Java. To stick with something SAP-provided, I used the signature of the equals() method in the if_xlf_comparable interface. The equals() method described here is not related to this specific interface – it can also be used with an interface you have written, or without any interface!

<h2>Full Code</h2>
```
*&---------------------------------------------------------------------*
*& Report  zss_demo_comparable
*&
*&---------------------------------------------------------------------*
*&
*&
*&---------------------------------------------------------------------*
REPORT zss_demo_comparable.

CLASS car DEFINITION.
  PUBLIC SECTION.
    METHODS:
      constructor IMPORTING producer TYPE string
                            model    TYPE string
                            year     TYPE i,
      equals IMPORTING object          TYPE REF TO object
             RETURNING VALUE(is_equal) TYPE abap_bool.

  PRIVATE SECTION.
    DATA:
      producer TYPE string,
      model    TYPE string,
      year     TYPE i.
ENDCLASS.

CLASS car IMPLEMENTATION.
  METHOD constructor.
    me->producer = producer.
    me->model = model.
    me->year = year.
  ENDMETHOD.

  METHOD equals.
    IF object EQ me.
      is_equal = abap_true.
      RETURN.
    ENDIF.

    IF object IS NOT BOUND.
      is_equal = abap_false.
      RETURN.
    ENDIF.

    TRY.
        DATA(other) = CAST car( object ).
        is_equal = xsdbool( me->producer EQ other->producer AND me->model EQ other->model AND me->year EQ other->year ).
      CATCH cx_sy_move_cast_error.
        is_equal = abap_false.
    ENDTRY.
  ENDMETHOD.
ENDCLASS.

CLASS truck DEFINITION.
  PUBLIC SECTION.
    METHODS:
      constructor IMPORTING producer TYPE string
                            model    TYPE string.

  PRIVATE SECTION.
    DATA:
      producer TYPE string,
      model    TYPE string.
ENDCLASS.

CLASS truck IMPLEMENTATION.
  METHOD constructor.
    me->producer = producer.
    me->model = model.
  ENDMETHOD.
ENDCLASS.

START-OF-SELECTION.
  DATA(car1) = NEW car( producer = 'Mercedes' model = 'E400' year = 2016 ).
  DATA(car2) = NEW car( producer = 'BMW' model = 'X5' year = 2013 ).
  DATA(car3) = NEW car( producer = 'Mercedes' model = 'E400' year = 2016 ).
  DATA car4 TYPE REF TO car.
  DATA(car5) = car1.
  DATA(truck1) = NEW truck( producer = 'CAT' model = 'CT660' ).

  " 2 different non-null Car objects with different attributes
  cl_demo_output=>write( |Car 1 and Car 2 are the same:   { car1->equals( car2 ) }| ).

  " 2 different non-null Car objects with the same attributes
  cl_demo_output=>write( |Car 1 and Car 3 are the same:   { car1->equals( car3 ) }| ).

  " 1 non-null Car object with 1 null Car object
  cl_demo_output=>write( |Car 1 is the same as Car 4:     { car1->equals( car4 ) }| ).

  " 2 different references to the same Car object
  cl_demo_output=>write( |Car 1 is the same as Car 5:     { car1->equals( car5 ) }| ).

  " Self as an argument
  cl_demo_output=>write( |Car 1 is the same as itself:    { car1->equals( car1 ) }| ).

  " 1 Car object with 1 Truck object
  cl_demo_output=>write( |Car 1 and Truck 1 are the same: { car1->equals( truck1 ) }| ).

  cl_demo_output=>display(  ).
```

