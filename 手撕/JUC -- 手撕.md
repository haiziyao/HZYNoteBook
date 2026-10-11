


#### 1. 循环打印 a b c

```java
// 使用 Semaphopre
public class Question1 {  
  
  
    public static void main(String[] args) {  
        Semaphore A = new Semaphore(1);  
        Semaphore B = new Semaphore(0);  
        Semaphore C = new Semaphore(0);  
        new Thread(() -> printLetter("a",A,B)).start();  
        new Thread(() -> printLetter("b",B,C)).start();  
        new Thread(() -> printLetter("c",C,A)).start();  
  
    }  
  
    static void printLetter(String s, Semaphore curr, Semaphore next) {  
        for (int i = 0; i < 10; i++) {  
            try{  
                curr.acquire();  
                System.out.print(s);  
                next.release();  
            }catch (Exception e) {  
                Thread.currentThread().interrupt();  
                return;  
            }  
        }  
    }  
}
```

```java
// 使用 reentrentLock 和 Condition

```