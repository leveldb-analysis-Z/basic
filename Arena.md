# Arena 메모리 할당자

LevelDB는 빈번한 메모리 할당 및 해제가 필요하며, 시스템의 new/delete 또는 malloc/free 인터페이스를 직접 사용하여 메모리를 적용 및 해제하면 많은 메모리 조각이 생성되어 시스템 **성능이 저하.** 따라서 LevelDB는 성능을 보장하기 위해 메모리를 관리하기 위해 **영역 메모리 할당자**를 구현

#### Memtable 마다 arena의 멤버 변수가 있다 -->  flush 할때  메모리 해제

                메로리 할당시
                2. leveldb(request_size 할당 ) --> 1. Arena(block_size만큼) --> system

                if  request_size > 남은 block_size
                    1. request_size > block_size(1/4)이면  --> request_size
                    2. 그냥 block_size 할당

                else request_size > block 
                    1. return requrest_size





```c++
#include <atomic>
#include <cassert>
#include <cstddef>
#include <cstdint>
#include <vector>

namespace leveldb
{
  class Arena
  {
  public:
    Arena();

    Arena(const Arena&) = delete;

    Arena& operator=(const Arena&) = delete;

    ~Arena();

    char* Allocate(size_t bytes);

    char* AllocateAligned(size_t bytes);

    size_t MemoryUsage() const
    {
      return memory_usage_.load(std::memory_order_relaxed);
    }

  private:
    char* AllocateFallback(size_t bytes);

    char* AllocateNewBlock(size_t block_bytes);

    char* alloc_ptr_; //현재 할당 가능한 메모리 블록의 여유 공간 헤드에 대한 포인터
    size_t alloc_bytes_remaining_; // 현재 메모리 여유 공간의 크기를 저장

    std::vector<char*> blocks_; // 할당된 모든 메모리 블록에 대한 포인터를 보유(배열)

    std::atomic<size_t> memory_usage_;// Arena가 차지하는 총 메모리 크기 저장
  };

// leveldb --> arena 메모리 할당

// inline 는 C와 C++ 모두에서 지원하는 언어 기능으로, 간단히 말해서 함수 호출의 오버헤드를 피하기 위해 컴파일 단계에서 인라인 함수가 호출되는 위치에서 함수 코드를 직접 확장하는 것입니다.
  inline char* Arena::Allocate(size_t bytes)
  {
    assert(bytes > 0); 

    if( bytes <= alloc_bytes_remaining_ ) // 원하는 메모리 <= 지금 여우 메모리 크기
    {
      char* result = alloc_ptr_; // 남은 공간의 헤드 point 주소
      alloc_ptr_ += bytes; // 시작 + 사용할 byte
      alloc_bytes_remaining_ -= bytes;  // 사용한 만큼 삭제
      return result;
    }
    return AllocateFallback(bytes);  // 부족하면 할당 
  }



//  if 상황

char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) { // 신청하려는 byte가   1/4넘었는지 
                                // kBlockSize=4096   즉 1024 
    char* result = AllocateNewBlock(bytes);  // 필요한 만큼만 할당
    return result;
  }
  alloc_ptr_ = AllocateNewBlock(kBlockSize); //넘었으니 4096 할당, 헤드 point 초기화
  alloc_bytes_remaining_ = kBlockSize; // 남은 공강도 초기화

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes; // 공간 나눠주기
  alloc_bytes_remaining_ -= bytes;
  return result;
}


// arena --> system
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);
  memory_usage_.fetch_add(block_bytes + sizeof(char*),
                          std::memory_order_relaxed);
  return result;
}


// 이름에서 알 수 있듯이 이 함수에 의해 할당된 메모리는 정렬
// 메모리 정렬은 CPU가 데이터를 가져오는 횟수를 줄이고 성능을 향상시키는 것
// viod* --> malloc, realloc,calloc,free등에 사용

char* Arena::AllocateAligned(size_t bytes) {

  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  
  
  static_assert((align & (align - 1)) == 0,
                "Pointer size should be a power of 2");
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align - 1);
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  size_t needed = bytes + slop;
  char* result;
  if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // AllocateFallback은 항상 정렬된 메모리를 반환합니다.
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align - 1)) == 0);
  return result;
}


}
```
