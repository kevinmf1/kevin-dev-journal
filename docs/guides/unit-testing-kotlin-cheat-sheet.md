---
sidebar_position: 2
title: "Unit Testing Kotlin (Cheat Sheet + Best Practice)"
description: "Catatan pribadi sebagai referensi cepat untuk unit testing di Kotlin Android, mencakup decision making, cheat sheet, dan best practice."
date: 2026-04-15
---

# 🧪 Unit Testing Kotlin (Android) — Personal Notes

Catatan ini dibuat sebagai referensi cepat agar tidak perlu mencari materi ulang. Fokus pada decision making saat menulis unit test.

---

## 🧠 Fundamental Concept

### 🔹 Real Object vs Mock

**Rule:**
- Class yang diuji → **REAL**
- Dependency → **MOCK**

Contoh:
- Test ViewModel → ViewModel real, UseCase mock
- Test Repository → Repository real, DataSource mock

---

### 🔹 Mock vs Spy (`spyk`)

#### Apa itu `spyk`?

`spyk` = **spy object (partial mock)**

Artinya:
> Object asli (real object) yang tetap menjalankan behavior aslinya, tapi bisa di-override sebagian.

#### Perbedaan

| Type | Behavior |
|------|----------|
| `mockk()` | Semua fake |
| `spyk()` | Real + sebagian bisa di-mock |

#### Contoh

```kotlin
class Calculator {
    fun add(a: Int, b: Int) = a + b
    fun multiply(a: Int, b: Int) = a * b
}

val calculator = spyk(Calculator())

every { calculator.add(1, 2) } returns 10

assertEquals(10, calculator.add(1, 2)) // mocked
assertEquals(6, calculator.multiply(2, 3)) // real
```

#### 🎯 Kapan pakai spyk?

Gunakan spyk kalau:

- Mau test logic asli
- Tapi ada method internal yang perlu dikontrol

Contoh:

- override API call
- override database
- tapi business logic tetap real

#### ⚠️ Rule Aman Pakai spyk

- Mulai dengan real object
- Mock semua dependency luar
- Kalau mentok karena method internal → baru pakai spyk
- Kalau ragu → jangan pakai dulu

#### 🚫 Hindari spyk kalau:

- Bisa pakai mock biasa
- Test jadi sulit dipahami
- Terlalu sering dipakai (code smell)

---

## 🔁 Flow & Coroutine Testing

### 🔹 runTest

Gunakan kalau:

- Ada suspend
- Ada Flow
- Ada delay / coroutine

```kotlin
@Test
fun testSomething() = runTest {
    val result = repository.getData()
    assertEquals("OK", result)
}
```

### 🔹 first() vs Turbine

| Case | Gunakan |
|------|---------|
| Flow 1 emission | `first()` |
| Flow banyak emission | Turbine |

### 🔹 Turbine

Gunakan untuk:

- Testing urutan state
- Multi emission
- Async flow

Contoh:

```kotlin
viewModel.state.test {
    assertEquals(Loading, awaitItem())
    assertEquals(Success, awaitItem())
}
```

---

## 🔍 Decision Rules (Cheat Sheet)

### 1. Real vs Mock

- Object diuji → real
- Dependency → mock

### 2. Real vs Spy

- Semua method aman → real
- Perlu kontrol sebagian → spyk

### 3. Mock vs Fake

- Perlu verify / response → mock
- Perlu behavior sederhana stabil → fake

### 4. Assert vs Verify

- Fokus hasil → assert
- Fokus interaksi → verify

### 5. Coroutine Test

- Ada coroutine → runTest
- Tidak → tidak perlu

### 6. Flow

- Simple → `first()`
- Complex → Turbine

### 7. LiveData

- Pakai LiveData → InstantTaskExecutorRule
- Tidak → skip

### 8. Stub

- Stub seperlunya
- Jangan over-stub

### 9. Test Structure

Gunakan AAA:

```kotlin
// Arrange
// Act
// Assert
```

### 10. Test Naming

Gunakan format:

```kotlin
fun `methodName, expected behavior when condition`()
```

Contoh:

```kotlin
fun `getCached, returns null when cache is empty`()
```

---

## 🧠 Best Practice

### 1. Test Behavior, bukan Implementation

Test:

- input
- output
- side effect

Bukan:

- variable internal
- function private

### 2. Satu Test = Satu Scenario

❌ Buruk:

- 1 test banyak scenario

✅ Bagus:

- 1 scenario = 1 test

### 3. Coverage bukan segalanya

Coverage tinggi ≠ test bagus

Yang penting:

- assertion kuat
- scenario lengkap

### 4. Repetition di test itu OK

Test readable > test DRY

### 5. Edge Case penting

Selalu test:

- null
- empty
- boundary
- true/false

### 6. Verify secukupnya

Gunakan verify hanya jika:

- interaksi penting

Jangan semua diverify → test jadi rapuh

### 7. Kalau test susah → cek design

Tanda code bermasalah:

- terlalu banyak mock
- butuh banyak spyk
- setup panjang

Solusi:

- pecah class
- inject dependency
- pisahkan logic

### 8. Tools sesuai kebutuhan

- MockK → mocking
- runTest → coroutine
- Turbine → Flow
- Truth → assertion
- InstantTaskExecutorRule → LiveData

### 9. Test membantu detect bug

Unit test membantu menemukan:

- bug logic
- dependency problem
- code smell

---

## 🚀 Summary (Super Cheat Sheet)

| Kondisi | Gunakan |
|---------|---------|
| object diuji | → REAL |
| dependency | → MOCK |
| partial control | → SPYK |
| Flow simple | → `first()` |
| Flow complex | → Turbine |
| coroutine | → runTest |
| fokus hasil | → assert |
| fokus interaksi | → verify |

---

## 🔥 5 Pegangan Utama

1. Test behavior, bukan internal detail
2. Class diuji real, dependency mock
3. Gunakan AAA & naming jelas
4. Coverage penting, tapi bukan tujuan
5. Kalau test susah → kemungkinan design code bermasalah
