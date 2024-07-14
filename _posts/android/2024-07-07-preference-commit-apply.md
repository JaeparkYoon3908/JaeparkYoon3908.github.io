---
title: Android Preference commit과 apply의 차이
author: jaepark
date: 2024-07-10 10:38:00 +0900
categories: [Android]
tags: [preference]
pin: false
img_path: '/assets/img'
---
> Android Preference에서 data를 추가하기 위해서 Editor의 commit(), apply() 두 가지 방법을 이용합니다. 
> Preference 코드를 작성하면서 추후에 왜 이런 코드를 짰는지에 대한 명확한 이유를 설명하기 위해서는 commit()과 apply()의 차이점에 대해 알고 갈 필요가 있습니다.
{: .prompt-info }
## commit()과 apply()의 차이
### commit()
commit()은 결괏값을 boolean 값으로 return 하며 순차적으로 진행이 됩니다. 만약 두 개 이상의 editor가 commit()을 했을 경우, 제일 마지막에 호출한 
commit()이 Preference에 반영되어 저장합니다. 내부 코드는 어떻게 구현이 되어있을까요?<br>
```java
@Override
public boolean commit() {
    long startTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    MemoryCommitResult mcr = commitToMemory();

    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
        if (DEBUG) {
            Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                    + " committed after " + (System.currentTimeMillis() - startTime)
                    + " ms");
        }
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```
내부 코드에서는 Preference의 값을 디스크에 write 명령을 내린 후 끝날 때까지 await()를 하여
write가 끝나면 그 결괏값을 boolean 값으로 return 하여 정상적으로 값이 반영됐는지 확인할 수 있습니다. 공식 문서에 나온 설명대로
write의 결괏값을 리턴 받을 수 있는 장점이 있지만, 메인 스레드에서 Preference를 commit() 했을 때 자칫 ANR이 걸릴 수 있다는 단점이 존재합니다.
### apply()
apply()는 commit()과는 달리 return type이 없으며 commit()과 달리 두 개 이상의 editor가 apply()를 했을 때 비동기 방식으로 일어남으로 원자성을 보장하지 못합니다.
마찬가지로 apply()의 내부 코드를 살펴봤습니다.<br>
```java
@Override
public void apply() {
    final long startTime = System.currentTimeMillis();

    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }

                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };

    QueuedWork.addFinisher(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}
```
내부 코드를 살펴보면 apply()는 별도의 Thread를 생성하여 비동기적으로 Preference의 값을 write하고 있습니다. 이러한 방식은 commit()과 달리 메인 스레드에서 
apply()를 했을 때 ANR이 발생할 우려가 없고 비동기 작업으로 진행되어 성능적으로 더 뛰어날 수 있다는 장점이 있지만, 
별도의 Thread 생성하여 write 하고 있기 때문에 commit()보다는 좀 더 무겁다는 단점이 있습니다.

## 개선방법
commit()과 apply()의 차이점을 살펴보면 Preference의 한계점을 알 수 있습니다.
1. 다중 스레드 환경에서 원자성을 보장하지 못합니다.
2. Preference write의 결괏값을 받기 위해 commit()을 사용하면 ANR이 발생할 가능성이 있습니다.
3. ANR을 피하기 위해 apply()를 사용하면 Preference write의 결괏값을 받지 못하며 무거운 스레드를 별도로 생성하여 실행합니다.

이러한 한계점을 개선하기 위해 Datastore로 마이그레이션 하는 방법이 있지만, 
Datastore로 마이그레이션이 어려운 상황이라면 적절하게 Coroutine을 사용하여 처리할 수 있습니다.

```kotlin
fun runJob() {
    viewModelScope.launch {
        if (setPreferenceData("data")) {
            //프리퍼런스 성공 시 동작
        }
    }
}

suspend fun setPreferenceData(value: String): Boolean {
    return withContext(Dispatchers.IO) {
        editor.putString("KEY", value)
        editor.commit()
    }
}
```

**참고**
- [https://developer.android.com/reference/android/content/SharedPreferences.Editor](https://developer.android.com/reference/android/content/SharedPreferences.Editor)
