Лабораторная работа №5 по ТиМП

Работу выполнил: Сакулин Даниил

I. Cоздаем CMakeLists.txt

1. Устанавливаем версию CMake и стандарт языка:

```
$ cat > CMakeLists.txt <<EOF 
cmake_minimum_required(VERSION 3.4)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
EOF
```

2. Устанавливаем имя проекта:

```
$ cat > CMakeLists.txt <<EOF
project(banking)
EOF
```

3. Добавляем .cpp файлы и указываем директорию, где искать включаемые файлы:

```
$ cat > CMakeLists.txt <<EOF
add_library(banking STATIC banking/Account.cpp banking/Transaction.cpp)
target_include_directories(banking PUBLIC banking/)
EOF
```

4. Делаем, чтобы изначально тесты были отключены:

```
$ cat > CMakeLists.txt <<EOF
option(BUILD_TEST "Build tests" OFF)
EOF
```

5. Добавляем модуль, в который будем заходить, если тесты будут включены. В самом модуле включаем тестирование для текущего репозитория, подключаем директорию с gtest, указываем файл, который будет собираться, и библиотеки, которые будут к этому файлу подключаться:

```
$ cat > CMakeLists.txt <<EOF
if(BUILD_TESTS)
enable_testing()
add_subdirectory(submodules/gtest)
add_executable(check tests/tests.cpp)
target_link_libraries(check banking gtest_main gmock_main)
add_test(NAME check COMMAND check)
endif()
EOF
```

II. Создаем tests.cpp

1. Объявляем mock-объекты:

```
$ cat > tests.cpp <<EOF
#include "Account.h"
#include "Transaction.h"
#include <gtest/gtest.h>
#include <gmock/gmock.h>
class AccountMock : public Account {
public:
AccountMock(int id, int balance) : Account(id, balance) {}
MOCK_CONST_METHOD0(GetBalance, int());
MOCK_METHOD1(ChangeBalance, void(int diff));
MOCK_METHOD0(Lock, void());
MOCK_METHOD0(Unlock, void());
};
class TransactionMock : public Transaction {
public:
MOCK_METHOD3(Make, bool(Account& from, Account& to, int sum));
};
EOF
```

2. Добавляем непосредственно модульные тесты:

```
$ cat > tests.cpp <<EOF
TEST(Account, Mock) {
AccountMock acc(1, 100);
EXPECT_CALL(acc, GetBalance()).Times(1);
EXPECT_CALL(acc, ChangeBalance(testing::_)).Times(2);
EXPECT_CALL(acc, Lock()).Times(2);
EXPECT_CALL(acc, Unlock()).Times(1);
acc.GetBalance();
acc.ChangeBalance(100); // throw
acc.Lock();
acc.ChangeBalance(100);
acc.Lock(); // throw
acc.Unlock();
}
TEST(Account, SimpleTest) {
Account acc(1, 100);
EXPECT_EQ(acc.id(), 1);
EXPECT_EQ(acc.GetBalance(), 100);
EXPECT_THROW(acc.ChangeBalance(100), std::runtime_error);
EXPECT_NO_THROW(acc.Lock());
acc.ChangeBalance(100);
EXPECT_EQ(acc.GetBalance(), 200);
EXPECT_THROW(acc.Lock(), std::runtime_error);
}
TEST(Transaction, Mock) {
TransactionMock tr;
Account ac1(1, 50);
Account ac2(2, 500);
EXPECT_CALL(tr, Make(testing::_, testing::_, testing::_))
.Times(6);
tr.set_fee(100);
tr.Make(ac1, ac2, 199);
tr.Make(ac2, ac1, 500);
tr.Make(ac2, ac1, 300);
tr.Make(ac1, ac1, 0); // throw
tr.Make(ac1, ac2, -1); // throw
tr.Make(ac1, ac2, 99); // throw
}
TEST(Transaction, SimpleTest) {
Transaction tr;
Account ac1(1, 50);
Account ac2(2, 500);
tr.set_fee(100);
EXPECT_EQ(tr.fee(), 100);
EXPECT_THROW(tr.Make(ac1, ac1, 0), std::logic_error);
EXPECT_THROW(tr.Make(ac1, ac2, -1), std::invalid_argument);
EXPECT_THROW(tr.Make(ac1, ac2, 99), std::logic_error);
EXPECT_FALSE(tr.Make(ac1, ac2, 199));
EXPECT_FALSE(tr.Make(ac2, ac1, 500));
EXPECT_TRUE(tr.Make(ac2, ac1, 300));
}
EOF
```

III. Настройка сборочной процедуры на travis

1. Указываем язык, платформу и т.д. В скрипте добавляем строк для включения тестов:

```
$ cat > .travis.yml <<EOF
language: cpp
os:
- linux
addons:
apt:
sources:
- george-edison55-precise-backports
packages:
- cmake
- cmake-data
script:
- cmake -H. -B_build -DBUILD_TESTS=ON
- cmake --build _build
- cmake --build _build --target test -- ARGS=--verbose
EOF
```

IV. Настройка Coveralls.io

1.Редактируем CMakeLists.txt:

```
option(COVERAGE "Check coverage" ON)
.................................................
if (COVERAGE)
target_compile_options(check PRIVATE --coverage)
target_link_libraries(check --coverage)
endif()
```

2. Редактируем .travis.yml:

```
before_install:
- pip install --user cpp-coveralls
.....................................................
after_success:
- coveralls --root . -E ".*gtest.*" -E ".*CMakeFiles.*"
```
