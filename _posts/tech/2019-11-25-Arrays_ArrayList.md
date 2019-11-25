---
title: "[JAVA] Arrays.asList() 삽질기!"
date: 2019-11-25 17:00:00
lastmod : 2019-11-25 17:00:00
categories: [JAVA]
tags: [JAVA, wasteOfTime]
sitemap :
  changefreq : daily
  priority : 1.0
---

# Arrays.ArrayList 삽질기..

## 배경
- Arrays.asList() 사용 중 add(), remove() 등의 메서드가 동작하지 못하고 다음과 같은 예외를 발생시키고 있었다.

~~~java
        List<Integer> integers = Arrays.asList(1,2,3);
        integers.add(4); //java.lang.UnsupportedOperationException 발생!
~~~

<br/>

## 삽질 과정
### 1. Arrays.asList() 를 먼저 확인해봤다.
~~~java
    /**
     * Returns a fixed-size list backed by the specified array.  (Changes to
     * the returned list "write through" to the array.)  This method acts
     * as bridge between array-based and collection-based APIs, in
     * combination with {@link Collection#toArray}.  The returned list is
     * serializable and implements {@link RandomAccess}.
     *
     * <p>This method also provides a convenient way to create a fixed-size
     * list initialized to contain several elements:
     * <pre>
     *     List&lt;String&gt; stooges = Arrays.asList("Larry", "Moe", "Curly");
     * </pre>
     *
     * @param <T> the class of the objects in the array
     * @param a the array by which the list will be backed
     * @return a list view of the specified array
     */
    @SafeVarargs
    @SuppressWarnings("varargs")
    public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }
~~~

### 2 음.. ArrayList를 보니 살짝 멘붕이 왔다.. ArrayList는 분명히 아래와 같이 add, remove 등등을 override하고 있었기 때문이다.
~~~java
     /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
     /**
     * Removes the element at the specified position in this list.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).
     *
     * @param index the index of the element to be removed
     * @return the element that was removed from the list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
~~~

### 3. 자바는 다형성을 제공하기 때문에 분명히 오버라이딩한 ArrayList의 메서드가 호출됐어야 하는데 아래 AbstractList의 add, remove가 호출되고 있었다!

~~~java
    /**
     * {@inheritDoc}
     *
     * <p>This implementation always throws an
     * {@code UnsupportedOperationException}.
     *
     * @throws UnsupportedOperationException {@inheritDoc}
     * @throws ClassCastException            {@inheritDoc}
     * @throws NullPointerException          {@inheritDoc}
     * @throws IllegalArgumentException      {@inheritDoc}
     * @throws IndexOutOfBoundsException     {@inheritDoc}
     */
    public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }

    /**
     * {@inheritDoc}
     *
     * <p>This implementation always throws an
     * {@code UnsupportedOperationException}.
     *
     * @throws UnsupportedOperationException {@inheritDoc}
     * @throws IndexOutOfBoundsException     {@inheritDoc}
     */
    public E remove(int index) {
        throw new UnsupportedOperationException();
    }
~~~


### 4. 그래서 처음으로 돌아와 Arrays.asList()가 반환한 ArrayList를 확인해 봤다. 아뿔싸 그 ArrayList는 우리가 흔히 사용하던 java.util 내 ArrayList가 아니었다. 정확히 아래와 같이 Arrays내에 정의돼있는 inner class였다!
~~~java
    /**
     * @serial include
     */
    private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable
    {
        private static final long serialVersionUID = -2764017481108945198L;
        private final E[] a;

        ArrayList(E[] array) {
            a = Objects.requireNonNull(array);
        }

        @Override
        public int size() {
            return a.length;
        }

        @Override
        public Object[] toArray() {
            return a.clone();
        }

        @Override
        @SuppressWarnings("unchecked")
        public <T> T[] toArray(T[] a) {
            int size = size();
            if (a.length < size)
                return Arrays.copyOf(this.a, size,
                                     (Class<? extends T[]>) a.getClass());
            System.arraycopy(this.a, 0, a, 0, size);
            if (a.length > size)
                a[size] = null;
            return a;
        }

        @Override
        public E get(int index) {
            return a[index];
        }

        @Override
        public E set(int index, E element) {
            E oldValue = a[index];
            a[index] = element;
            return oldValue;
        }

        @Override
        public int indexOf(Object o) {
            E[] a = this.a;
            if (o == null) {
                for (int i = 0; i < a.length; i++)
                    if (a[i] == null)
                        return i;
            } else {
                for (int i = 0; i < a.length; i++)
                    if (o.equals(a[i]))
                        return i;
            }
            return -1;
        }

        @Override
        public boolean contains(Object o) {
            return indexOf(o) != -1;
        }

        @Override
        public Spliterator<E> spliterator() {
            return Spliterators.spliterator(a, Spliterator.ORDERED);
        }

        @Override
        public void forEach(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            for (E e : a) {
                action.accept(e);
            }
        }

        @Override
        public void replaceAll(UnaryOperator<E> operator) {
            Objects.requireNonNull(operator);
            E[] a = this.a;
            for (int i = 0; i < a.length; i++) {
                a[i] = operator.apply(a[i]);
            }
        }

        @Override
        public void sort(Comparator<? super E> c) {
            Arrays.sort(a, c);
        }
    }
~~~

<br/>

## 결론
### 1. Arrays내 ArrayList는 inner class 였고 AbstractList를 상속하고 있었다. 그렇기 때문에 재정의 하지 않은 add(), remove()는 AbstractList내 add(), remove()를 호출하게 됐던 것이다. (이름만 같은 전혀 다른 클래스!)
~~~java
     /**
     * @serial include
     */
    private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable
    {
      ...
    }
~~~

### 2. set메서드가 구현되어있기 때문에 구글에서 작성한 ImmutableList 클래스와는 차이가 있었다. Arrays.asList()의 반환형인 Arrays내 ArrayList는 set을 재구현했기 때문에 set 연산이 가능해진다. 이 점으로 인해 Arrays내 ArrayList가 Immutable 하다고 정의하는 건 무리가 있다고 생각된다.
~~~java
  //이건 ImmutableList의 set입니다.
  /**
   * Guaranteed to throw an exception and leave the list unmodified.
   *
   * @throws UnsupportedOperationException always
   * @deprecated Unsupported operation.
   */
  @CanIgnoreReturnValue
  @Deprecated
  @Override
  public final E set(int index, E element) {
    throw new UnsupportedOperationException();
  }
 
  //이건 Arrays내 ArrayList의 set입니다.
  @Override
  public E set(int index, E element) {
    E oldValue = a[index];
    a[index] = element;
    return oldValue;
  }

~~~ 

