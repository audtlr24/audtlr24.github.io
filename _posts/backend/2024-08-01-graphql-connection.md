---
layout: post
title: GraphQL Connection
date: 2024-08-01 12:00:00 +0000
categories: backend
---

## GraphQL Connection

### Intro
Graphql 에서 여러 데이터를 가져오는 query 에서 pagination 을 다루는 connection 의 내용을 정리하고자 함

### Pagination
pagination 은 데이터를 list 형태로 조회하는 케이스에서 전체 데이터를 긁어오지 않고 일부의 데이터만 가져와서 보여주는 방법이 필요하고, 이를 흔히 pagination 이라고 표현
가장 쉬운 게시판을 예로 들었을때, 1page, 2page... 등 page 를 누르면 해당 게시판에서 다음 순서의 게시글 목록들을 볼 수 있는 것도 pagination 구현

### OffSet-based Pagination
쉽게 page number 를 Input 으로하고 이를 기준으로 data 를 쿼리해서 전달하는 방식
크게 2가지 문제가 있는데,
- 페이지 요청 사이에 데이터 변경이 있을때, 중복 데이터 노출
	- 다음 페이지로 넘어갔는데 새로 추가된 게시글이 있으면 이전 페이지 내용 일부가 다음페이지에 있게됨
- 대부분 RDBMS 에서는 OFFSET 쿼리 퍼포먼스 이슈가 있음
	- offset 이 큰 값인 경우 DB 에서는 offset + 원하는 갯수 만큼의 데이터를 얻어오고 앞에 값들을 잘라낼수 밖에 없음 (정렬 때문)
이 문제들을 해결하면서 특히 요새 모바일에서 많이 사용 되는 "infinite scroll" 같은 시스템에서 
 
## Cursor Connection Pagination

위의 단점을 극복하고자 Connection 이라는 패턴을 제안함
1. 데이터 쿼리시 데이터와 함께 "cursor" 라는 해당 데이터 위치 정보를 함께 전달
2. 데이터 쿼리에서 이전 데이터의 cursor 를 함께 넣어서 요청하면, 해당 데이터 이후의 데이터를 기준으로만 탐색

Example)
```json
{
  user {
    id
    name
    friends(first: 10, after: "opaqueCursor") {
      edges {
        cursor
        node {
          id
          name
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
}
```

위에가 spec 문서에서 보여주는 example 의 connection 구성인데 하나씩 살펴보면

### Edges
여러 데이터가 실제로 들어있는 list
- Node
	- 실질적으로 원하는 데이터 Object
- Cursor
	- 데이터의 위치 정보를 나타내는 값
	- 정의하기 나름이지만 cursor 라는게 어떤 특정한 의미를 갖지 않는 다는 것을 나타내기 위해 base64 encoding 을 사용함

### PageInfo
이후 혹은 이전의 page 정보, page 유무 등을 표현하는 메타데이터
- hasNextPage, hasPreviousPage
- optional) startCursor, endCursor

### Argument
입력으로 넣는 값
- ContinuationInfo
	- 기준이 되는 cursor, 탐색 방향, entity 수 의 정보가 포함되고 탐색방향에 따라 forward, backward 로 구분됨
	- forward 는 first(정방향으로 가져올 데이터 수), after (기준 cursor) 
	- backward 는 last(역방향으로 가져올 데이터 수), before (기준 cursor)


## Implementation

```typescript
export async function Connection<Entity, RawEntity = Entity>(  
  qb: SelectQueryBuilder<Entity>,  
  cursorClass: CursorConstructor<Entity, RawEntity>,  
  continuationInfo: ContinuationInfo,  
  fnFilter: ConnectionFilter<RawEntity> = (e) => Promise.resolve(e),  
): Promise<IConnectionModelEx<RawEntity>> {  
  const edges = await SelectEdges(qb, cursorClass, continuationInfo, fnFilter);  
  const page = await PageInfo(qb, cursorClass, edges, fnFilter);  
  return ConnectionModelImpl.of(edges, page);  
}
```

```typescript
async function SelectEdges<Entity, RawEntity = Entity>(  
  qb: SelectQueryBuilder<Entity>,  
  cursorClass: CursorConstructor<Entity, RawEntity>,  
  continuationInfo: ContinuationInfo,  
  fnFilter: ConnectionFilter<RawEntity>,  
): Promise<IEdgeModelEx<RawEntity>[]> {  
  if (continuationInfo.numberOfEntities <= 0) return [];  
  const limit = _.max([continuationInfo.numberOfEntities, 10]);  
  
  const entities = [];  
  let applyAfter = !!continuationInfo.cursor;  
  let cursor = deserializeCursor(cursorClass, continuationInfo.cursor);  
  while (true) {  
    const query = cursor.query(continuationInfo.isForward);  
    const _entities = await pipe(  
      applyAfter ? query.after(qb) : qb,  
      (_qb) => query.orderBy(_qb).take(limit),  
      (_qb) => (cursor.withDeleted ? _qb.withDeleted() : _qb),  
      (_qb) => (cursor.usingRawQuery ? _qb.getRawMany() : _qb.getMany()),  
    );  
    if (_entities.length === 0) break;  
  
    const filtered = await fnFilter(_entities);  
    entities.push(...filtered);  
  
    if (entities.length >= continuationInfo.numberOfEntities) break;  
  
    cursor = cursorClass.fromEntity(_entities[_entities.length - 1]);  
    applyAfter = true;  
  }  
  
  const _entities = entities.slice(0, continuationInfo.numberOfEntities);  
  const sorted = continuationInfo.isForward ? _entities : _entities.reverse();  
  
  return sorted.map((e) => EdgeImpl.of(e, serializeCursor(e, cursorClass)));  
}
```

```typescript
async function PageInfo<Entity, RawEntity>(  
  qb: SelectQueryBuilder<Entity>,  
  cursorClass: CursorConstructor<Entity, RawEntity>,  
  edges: IEdgeModel<RawEntity>[],  
  fnFilter: ConnectionFilter<RawEntity>,  
): Promise<ConnectionPageInfoModel> {  
  if (edges.length === 0) return ConnectionPageInfoModel.empty();  
  
  const startCursor = edges[0].cursor;  
  const endCursor = edges[edges.length - 1].cursor;  
  
  const previous = await SelectEdges(qb, cursorClass, { numberOfEntities: 1, cursor: startCursor, isForward: false }, fnFilter);  
  const next = await SelectEdges(qb, cursorClass, { numberOfEntities: 1, cursor: endCursor, isForward: true }, fnFilter);  
  
  return ConnectionPageInfoModel.of(startCursor, endCursor, previous.length === 1, next.length === 1);  
}
```