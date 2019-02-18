---
title: cpp17 An apprach to std::variant deserialization
date: 2019-01-30 00:00:00 Z
categories:
- blog
tags:
- cpp
- c++
- cpp17
- c++17
layout: post
---

## If you have serialized it, that's it 

[gcc.godbolt.org link](https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAKxAEZSAbAQwDtRkBSAJgCFufSAZ1QBXYskwgA5NwDMeFsgYisAag6yAwskEF8LAhuwcADAEE5CpSszqtCwniYM8AL0zEA%2Bi91HTFrnlFZTUNTQIATwAHTE8CYiZCQT9zS2CbO00ANyZiJwMUsxFBBWBVFiYAW0xBKKYJVV10DT5U8wJMSqjmDszImIrq1QAVQrQWXVURBQIADjjVACoAM2JUSs8AIwUIYfUuADZSVXHJ6YN5giXBcQBKFv9/AHon1WJMZDESrNsOrp7bHh/p1MAYmAQ8KgWI92p1uuDbGF%2BqCqrYAGpmABKAEkzAA5YbHZGDWwACSJ0RR1QAdLSRmMoWcZpclqt1lsFJ4cnlWAQIBicfi9twDlhdMdznMFgosAAPUj%2BVRK5Uq1Vq9Uq05XSUsxY3ZC3dQAdlaZiVeGWqggMswsrsABENPbVCZDRwTYrlaTVAgHuYVfqHao2RtttCtKSjBAEMd9fdZKaVWKrk6fX6zcr3gQxCxGuJ00r3c7MAxBIDLVrbVFiFaSu5UMtadTdm7ZNgXW6Pf7M5hs8RcyGOSwubl8oYtALcQTjsMm1Hk8cbXaALSqeh5g0F42O1JGncWWH/BF9Skkpv0tv%2BLVTZkLFZrUOc96fYjfTAQbljsKz2lGfYHHIGGOa8dTvONjUTG8LmlFg5S3QNU0HMMwlA8dsGtWDbVjO54PEIMkM5T9eTCIiCi0H9qSMGc5zbD9nEXTD5Q3eNIKzHMN3TIsYTMF5VCXRpMAARxEUEGiBboQTBCEoW4v54V6JFT1RVQ0VHXlYz0EAQDrWICHPbFkkvcwQNvK573ZMMRx5AxPDwQR%2BTUgxVEAiVTL4xiFW7DVvJ8tUTOgsz9QYuVPDLYTRMwMIDJo7BO0gpgRCIRoOiiIMOAAVj4Q4MvtCAEqSuzjny1BVAIOLPXNS0MLlB1Uzs8qvOVBDZGdAjw00LAlGRCAyqjAhsM3BMKuVQCg3HSDCz3Ya2P7F0ty4oa2gzJotIcCFnDcDxvDs8dNBmPwTV0TAoggAzjlIghPGcDp%2B3BPAfjiKLBGOVTrLQ90%2BD3W45ymxaMxm3N9U4qalrkgETwGVFz1GIyzH8qUzLa2zGLCui3u/GL/xck5GW1Ny9TuCDPVQ9y4L%2BpVmtah8hxQ0yoyXAaWM9Sng2pyyLts%2BzhqVbGGY3Y5KiYABrWIl1CoSRMUSKtA5nSuRIxzdoovxL1o24me7AGOL%2Brilt0YgRGQK4zQ%2B4m3KYYHHT%2B/XDauHgie7SVaAOBZNkt9MbaN1RNAdjNJVkLgFk4HXftNZ5XmQPBxCYNcjQOVRV1oWR0qDAAWWh3M96SJn8MHj3h3UkefL57ulzRKihVBdARY4zGOARvajYaLrCCuWCrgga9UOvVAbzQ/xFYDcaghGlg1niI6j5AY64VPU4T2OjSDA4l4ULPIRzw95NsAu7yRpcwtbyvq46Wv6%2BOfvaObxWj/bk/MDP3uL4Hw4h4mPGArHzjzF4wWWBEZwjQADuhBkAID4sCaoUkN7hxOFPGOScU6rlmEvVMqdZiZ3iLbDeyRjLDxJuZR8w5BAgIIGAtGX4tBtw7l3HufcX4AVyJ5DMvlWF%2BXwfjcCptHZuSXLhZA%2BE2YKFpgFemHlmLwVIWAq0S4GosJOEwMsLoQDc1UNQ%2B%2BaitwBjwohIR7V1Gdw6FGSojMtEjVyEGSoZilSbHeMLMx08lG0BUY1JUZoLbk1VCzNqYQLC0SYKYzxKpuRBg8RNZUtjMD2KCQopRXAXHyKVPbN2MSKY6JaqzCywitB8FopsQJ4SeYWNTCkwpqhInRPCY42wsgElqh9sHMp3i9FhEvuhZABTVEhNTI01RFShbzRBgef6vZ2JAxDvuKQtxGDSHSlIUgLBpAmHmagaQ/deD8EaKIcQiJAi0HmQQJZUzplCxAKnLg1JZipwOLIWY6V0rnOdrMFeRwGDSFTvMxZUhlmkFWVIeZggQAmFIIc75UzSBwFgEgNAXQ8AMA8OQSgMKohwo8CAXIlRgCzC4MC5YcKbqAogJsI5pAwy5AiNIfZpAYVQIIAAeRYAwClYLSBYD/sAeFJL8DPghD8QFLLbQfESpIKQVKZglhJS4Wx5LNAYBFVS%2BIQIjnTOYGwFA/B%2BCMDwJsQFkBpmoCiNnfly4mhOk4Bs3gXAY7LjpYIBOlRkBRBEGakgHRZTLkqLIAF2yJB0BVbMz5JK/lYgALKqGAMgARsxqRcCtLgQgJB9iyHXLK2F8KaxyFoIadZWUeAHOVSckA6UjTUloOlWQBxaBz2LbMZNAdU4zKkB80glQ6AmGBV8n5fyAVApBQWxtXB5mttoO2hZQbpD5rBerUgPxXwbzOUAA%3D%3D%3D)

```cpp
#include <cstdint>
#include <initializer_list>
#include <type_traits>
#include <variant>
using namespace std;

template <typename T>
const uint8_t *from_bin(T &, const uint8_t *src);

// recursive template implementation

template <typename VARIANT, typename H, typename... T>
const uint8_t *from_bin_variant(VARIANT &dest, uint8_t index,
                                const uint8_t *src) {
  if (index == 0) {
    H h;
    src = from_bin<H>(h, src);
    dest = h;
    return src;
  } else if constexpr (sizeof...(T) > 0) {
    return from_bin_variant<VARIANT, T...>(dest, index - 1, src);
  }
}

template <typename... T>
const uint8_t *from_bin_recursive(variant<T...> &val, const uint8_t *src) {
  uint8_t index;
  src = from_bin<uint8_t>(index, src);
  src = from_bin_variant<variant<T...>, T...>(val, index, src);
  return src;
}

// index sequence implementation

template <typename Variant, std::size_t... Is>
const uint8_t *from_bin_variant_is(Variant val, uint8_t index,
                                   const uint8_t *src, index_sequence<Is...>) {
  auto step = [&](auto is, auto t) {
    if (index == is) {
      src = from_bin<decltype(t)>(t, src);
      val = t;
    }
    return 0;
  };

  std::initializer_list<int>{step(Is, variant_alternative_t<Is, Variant>{})...};
  return src;
}

template <typename... T>
const uint8_t *from_bin_indexseq(variant<T...> &val, const uint8_t *src) {
  uint8_t index;
  src = from_bin<uint8_t>(index, src);
  src = from_bin_variant_is(
      val, index, src, make_index_sequence<variant_size_v<variant<T...>>>());
  return src;
}

struct A {
  uint8_t a;
};
struct B {
  uint16_t b;
};
struct C {
  uint32_t c;
};

// circa 176 - 135 = 41 instructions
template const uint8_t *from_bin_recursive<monostate, A, B, C>(
    variant<monostate, A, B, C> &, const uint8_t *);
// circa 244 - 177 = 67 instructions
template const uint8_t *from_bin_indexseq<monostate, A, B, C>(
    variant<monostate, A, B, C> &, const uint8_t *);

// manual switch implementation
// circa 135 - 87 = 48 instructions
const uint8_t *from_bin_switch(variant<monostate, A, B, C> &var,
                               const uint8_t *src) {
  uint8_t index;
  src = from_bin<uint8_t>(index, src);
  switch (index) {
    case 0:
      monostate m;
      src = from_bin<monostate>(m, src);
      var = m;
      break;
    case 1:
      A a;
      src = from_bin<A>(a, src);
      var = a;
      break;
    case 2:
      B b;
      src = from_bin<B>(b, src);
      var = b;
      break;
    case 3:
      C c;
      src = from_bin<C>(c, src);
      var = c;
      break;
  }

  return src;
}
```