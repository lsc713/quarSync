---
title: BFS 알고리즘 정리
date: 2026-02-05
tags: [algorithm, bfs]
---

# BFS 알고리즘 정리

너비 우선 탐색 문제 풀이 패턴입니다.

## 기본 구조
```java
Queue<int[]> queue = new LinkedList<>();
boolean[][] visited = new boolean[N][M];

queue.offer(new int[]{0, 0});
visited[0][0] = true;

while (!queue.isEmpty()) {
    int[] cur = queue.poll();
    // 로직
}
```


깊이 별 더하기
```
bfs(int x,int y){
 
 ...
 while(!q.isEmpty()){
   int size = q.size();
   for(int i = 0 ; i <size;i++){
       int cur = q.poll();
       visited and so on...
   }
   size++;
  }
  ...
}
```

거꾸로 bfs돌리기
```


시작점이아니라 끝나는점을 기준으로 bfs를 돌리고
grid배열외에
dist 배열을 -1로 초기화하고
q에 넣을 때 0으로 넣게되서 bfs를 돌리면
시작점까지 거리가 나옴.
최소거리를 알 수 있고, 이를 통해 결과를 반환하면 도달할 수 없는 곳은 -1로 초기화되어 바뀌지 않아 해를 구하기 쉽다.
```