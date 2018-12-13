# コピー

クラスにコピーを正しく実装するためには、まずコピーが何であるかを理解しなければならない。

## 普通のコピー

C++を書くユーザーは、クラス型のオブジェクトを使うとき、クラスが普通の型(regular type)のように振る舞うことを期待している。この普通には様々な意味がある。


`int`型の変数をコピーするとき、コピー先の変数はコピー元の変数と等しくなる。

~~~c++
int source = 42 ;
int destination = x ;
~~~

この例では変数`destination`は変数`source`と等しくなる。`source == destination`は`true`となり、`destination`の値は`42`になる。

コピーの結果、コピー先の変数は値が書き換えられる。コピー元の変数は変わらない。上の例で、変数`source`が勝手に別の値になることは「普通」はない。

我々が普通にコピーと認識しているものは、C++の文法的にはコピー構築とコピー代入に分けることができる。

~~~c++
int source = 42 ;
// コピー構築
int b = source ;
int c(source) ;
int d{source}

// 変数e
int e ;
// コピー代入
e = source ;
~~~

ユーザーは普通、コピー構築とコピー代入のコピーが両方とも同じ挙動をすると期待している。コピー構築とコピー代入のどちらか片方が使えるならば、もう片方も使えるべきで、そのコピーの挙動は同じであるべきだ。

コピー代入にはコピーの普通に加えて、さらにユーザーが代入に期待する普通がある。

代入式を評価した結果は、代入されるオブジェクトへのlvalueリファレンスになる。

~~~cpp
int main()
{
    int x, y, z ;
    // x, y, zに0を代入
    x = y = z = 0 ;
}
~~~

これはまず`z = 0`が評価される。変数`z`の値は`0`になり、式を評価した結果の値は`z`へのlvalueリファレンスだ。なので、`y = z = 0`というのは、`y = (z=0)`となる。`z=0`については`z`であるので、`y = z`となる。ここでの`z`は`0`を代入された後の`z`なので、値は`0`だ。その結果変数`y`の値は`0`になる。変数`x`の場合も同様だ。

以下のような例も見てみよう。

~~~cpp
int main()
{
    int x ;
    (x = 0) = 1 ;
}
~~~

これは`(x = 0)`の結果に`1`を代入している。`x=0`の結果は`x`なので、`x`には`0`が代入された後に`1`が代入される。結果として`x`の値は`1`になる。

## コピーコンストラクター

コピー構築の場合、コピーコンストラクターが呼ばれる。

~~~c++
struct Value
{
    // コピーコンストラクター
    X( const X & source )
    { }
} ;

int main()
{
    Value source ;
    // コピーコンストラクターを呼ぶ
    Value b = source ;
    Value c(source) ;
    Value d{source} ;
}
~~~

コピーコンストラクターは`クラス型へのlvalueリファレンス型`を引数にとる`コンストラクター`だ。

~~~cpp
struct X
{
    X( const X & source ) { }
} ;
~~~

引数は通常は`constなlvalueリファレンス型`だが、`非constなlvalueリファレンス型`を引数に取るコンストラクターも`コピーコンストラクター`となる。

~~~cpp
struct X
{
    X( X & source ) { }
} ;
~~~

ただし、非constなlvalueリファレンス型を引数に取るコピーコンストラクターは通常は使わない。なぜならば、コピーの結果、コピー元が書き換えられるような挙動は不自然だからだ。

~~~cpp
struct funny_number
{
    int n ;
    funny_number( int n = 0 )
        : n(n) { }
    funny_number( funny_number & source )
        : n( source.n )
    {
        source.n = 0 ;
    }
} ;

int main()
{
    funny_number a = 1 ;
    // コピー
    funny_number b = a ;
    // a == 1
    // b == 0
}
~~~

このおかしな`funny_number`のコピーコンストラクターはコピー元を0に書き換えてしまう。このコードは完全に合法なC++のコードだが、このようにコピーコンストラクターを実装するのはおすすめできない。なぜならば、ユーザーはコピーについて上で示したような意味を普通だと想定しているため、普通から外れるような型はユーザーのあてが外れてしまうからだ。

## コピー代入演算子

コピー代入演算子は`クラス型へのlvalueリファレンス型`を引数に取る`operator =`のオーバーロードだ。

~~~cpp
struct X
{
    X & operator = ( const X & source )
    {
        return *this ;
    }
} ;
~~~

コピーコンストラクターと同じく、コピー代入演算子の引数は非constなlvalueリファレンスでもよい。ただし、ユーザーの期待する普通にはそぐわない結果になる。

コピー代入演算子の戻り値の型はクラス型への非constなlvalueリファレンスでなくてもよい。ただし、その場合もユーザーの期待にそぐわないことになる。

~~~c++
struct X
{
    void operator = ( const X & source ) { }
} ;

int main()
{
    X a, b, c ;
    // OK
    a = b ;
    // エラー
    a = b = c ;
}
~~~

`a = b = c`は、クラス`X`のコピー代入演算子の戻り値の型が`void`なので動かない。ユーザーは普通、これが動くことを期待している。ユーザーの普通の期待に答えるためにはクラスへの非constなlvalueリファレンスを返さなければならない。

## コピーの挙動

クラスのコピーは何をすればいいのだろうか。クラスにコピーコンストラクターとコピー代入演算子を書かない場合、デフォルトのコピーコンストラクター、コピー代入演算子が生成される。

デフォルトのコピーは、クラスのデータメンバーをそれぞれコピーする。

~~~cpp
struct Point
{
    int x ;
    int y ;
    int z ;
} ;

int main()
{
    Point a{1,2,3} ;
    Point b = a ;
    Point c ;
    c = a ;
}
~~~

上記のコードは、以下のように書いたのと同じだ。

~~~c++
Point b{ a.x, a.y, a.z } ;
Point c ;
c.x = a.x ;
c.y = a.y ;
c.z = a.z ;
~~~

つまり、以下のようなコピーコンストラクターとコピー代入演算子を書いたのと同じだ。

~~~cpp
struct Point
{
    int x ;
    int y ;
    int z ;

    Point( const Point & r )
        : x(r.x), y(r.y), z(r.z)
    { }

    Point & operator = ( const Point & r )
    {
        x = r.x ;
        y = r.y ;
        z = r.z ;
    }
} ;
~~~

では`std::vector`のコピーはどうなるだろうか。

~~~cpp
int main()
{
    std::vector<int> v = {1,2,3,4,5} ;
    std::vector<int> w = v ;
    // wは{1,2,3,4,5}
}
~~~

`std::vector`をコピーした場合、その値がコピーされる。

自作の`vector`のコピーはどのように実装すればいいだろうか。デフォルトのコピーに任せてもいいのだろうか。デフォルトのコピーを使う場合、コピーコンストラクターは以下のように書いたものと同じだ。

~~~c++
template < typename T, typename Allocator = std::allocator<T> >
class vector
{
    // ... その他のメンバー
private :
    pointer first = nullptr ;
    pointer last = nullptr ;
    pointer reserved_last = nullptr ;
    allocator_type alloc ;

public :
    // コピーコンストラクター
    vector( const vector & r )
        : first( r.first ), last( r.last ),
        , reserved_last( r.reserved_last ),
        , alloc( r.alloc )
    { }
} ;
~~~

これは問題だ。以下のコードを考える。

~~~c++
int main()
{
    vector<int> v{1} ;
    vector<int> w = v ;
    // wのデストラクターが呼ばれる
    // vのデストラクターが呼ばれる
}
~~~

`w = v`で、`v`のデータメンバーの値がそれぞれ`w`のデータメンバーにコピーされる。

main関数を抜けるので、構築の逆順に変数が破棄される。この場合`w`が先に破棄される。破棄にあたっては`w`のデストラクターが呼ばれる。

この場合、`w`のデストラクターは、

1. ポインター`first`が指すオブジェクトのですトラクターを呼び出す
2. ポインター`first`の指す生のストレージを解放

する。

次に`v`が破棄される。`v`のデストラクターは`w`のデストラクターと全く同じことをする。ただし、ポインター`first`の指すオブジェクトはすでにデストラクターが呼び出されているし、ポインター`first`の指す生のストレージも開放されている。

すでにデストラクターを呼び出したオブジェクトに対してもう一度デストラクターを呼び出した場合の挙動は未定義だ。すでに解放したストレージを指すポインターに対してもう一度ストレージの解放した場合の挙動は未定義だ。したがって、このプログラムの挙動は未定義となる。

コピー代入も同じ問題を抱えている。しかも別の問題まである。例えば以下の例を見てみよう。

~~~cpp
int main()
{
    vector<int> v = {1,2,3} ;
    vector<int> w = {4,5,6} ;
    w = v ;
}
~~~

変数`w`はまず要素を保持するためのメモリを動的確保する。その後、`w`に`v`が代入されるわけだが、このとき`w`が動的確保したメモリを指すポインターの値が上書きされてしまう。`w`が破棄されるとき、`w`が元々持っていた要素は破棄されなくなり、ストレージも開放されなくなる。


## 所有するクラス

この問題は「所有」という考え方を使うと解決できる。

問題を簡単にするために、以下のようなクラスを考えよう。

~~~
class own
{
private :
    int * ptr ;
public :
    own( int value = 0 )
        : ptr( new int( value ) )
    { }
    ~own()
    { delete ptr ; }

} ;
~~~

このクラスはコンストラクターで動的メモリ確保をし、デストラクターでメモリの解放をする。

コピーコンストラクターとコピー代入演算子は定義していないので、デフォルトのコピーが使われる。

デフォルトのコピーを使うことを明示する方法もある。`= default`を使うのだ。
~~~cpp
class own
{
    // その他のメンバー
public :
    own( const own & ) = default ;
    own & operator ==( const own & ) = default ;
}
~~~

コピーコンストラクター、コピー代入演算子となる宣言に`= default`を使うと、デフォルトのコピー実装を使うということを明示的に宣言したことになる。この文法はややわかりにくいが、こういうものだと思って覚えておこう。

このようなクラスを「デフォルトのコピー」でコピーしたとき、コピーされるのはポインターの値だ。ポインターが参照する先は同じだ。

この場合、クラスはポインターの参照するオブジェクトを所有していると考えることができる。ポインターの値をコピーするということは、所有権を共有するということだ。所有権を共有していることを考慮しないまま、クラスのオブジェクトが破棄されたときにポインターの参照先まで破棄してしまうと、所有したつもりになっているクラスのオブジェクトが出来上がってしまう。

普通の型のように振る舞うコピーを実装するには、コピーの際に所有権を共有しない実装をする。具体的には、コピーのときに新しく動的メモリ確保し、値をコピーするのだ。

コピーコンストラクターは以下のようになる。

~~~c++
own( const own & r )
    : ptr( new int( *r.ptr ) )
{ }
~~~

今回の場合、コピー代入演算子で動的メモリ確保をする必要はない。なぜならば、コピー代入演算子が呼ばれたということは、いずれかのコンストラクターがすでに呼ばれていて、動的メモリ確保はされているからだ。

~~~c++
own & operator = ( const own & r )
{
    *ptr = *r.ptr ;
    return *this ;
}
~~~

このコードには少し問題がある。

## もう少し複雑な所有するクラス

同じく所有するクラスだが、もう少し複雑な例を考えよう。

~~~cpp
template < typename T >
class dynamic_array
{
private :
    T * first ;
    T * last ;
public :
    dynamic_array( std::size_t size = 0 )
        : first( new T[size]), last( first + size )
    { }
    ~dynamic_array()
    { delete[] first ; }

    T & operator [] ( std::size_t i )  const noexcept
    { return first[i] ; }
    std::size_t size() const noexcept
    { return last - first ; }
    T * begin() const noexcept
    { return first ; }
    T * end() const noexcept
    { return last ; }
} ;

int main()
{
    dynamic_array<int> a(10) ;
    a[0] = 1 ;
    a[1] = 2 ;
}
~~~

この`dynamic_array<T>`は`T`型の動的な配列クラスだ。配列のサイズは実行時に指定できる。

このようなクラスのコピーはどうなるだろうか。

コピーコンストラクターは簡単だ。コピー元と同じサイズの配列を動的確保し、要素をコピーすればいいだけだ。

~~~c++
dynamic_array( const dynamic_array & r )
    : first( new T[r.size()]), last( first + r.size() ) 
{
    std::copy( r.begin(), r.end(), begin() ) ;
}
~~~

コピー代入演算子でも、場合によっては動的メモリ確保が必要になる。現在所有しているメモリとは異なるサイズのオブジェクトからコピーする場合だ。

~~~cpp
int main()
{
    dynamic_array<int> a(5) ;
    dynamic_array<int< b(10) ;
    // aの所有するメモリはサイズ不足
    a = b ;
}
~~~

コピー元よりコピー先のほうがメモリが多い場合、つまり`b = a`の場合は動的メモリ確保をしないという実装もできるが、今回はサイズが違う場合は必ず動的メモリ確保をすることにしよう。

~~~c++
dynamic_array & operator == ( const dynamic_array & r )
{
    // サイズが違う場合
    if ( size() != r.size() )
    {
        // コピー処理
    }
    return *this ;
}
~~~

`new`したメモリは`delete`しなければならない。そこで、コピー代入演算子はまず自分の所有するメモリを`delete`してから`new`し、値をコピーすることになる。

~~~c++
dynamic_array & operator == ( const dynamic_array & r )
{
    if ( size() != r.size() )
    {
        // コピー先が所有しているメモリの解放
        delete first ;
        // コピー元と同じサイズの動的メモリ確保
        first = new T[r.size()] ;
        last = first + r.size() ;
        // コピー元の値をコピー
        std::copy( r.begin(), r.end(), begin() ) ;
    }
    return *this ;
}
~~~