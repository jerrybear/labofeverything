# Relationship Diagram

## 목적

- `influence-map.md`의 내용을 다이어그램으로 옮기기 쉬운 텍스트 형식으로 정리한다.
- 나중에 Mermaid, Excalidraw, Obsidian Canvas 같은 도구로 옮길 때 바로 활용할 수 있게 한다.

## Mermaid 초안

```mermaid
graph TD
  Hildegard[힐데가르트] --> Machaut[마쇼]
  Machaut --> Josquin[조스캥]
  Josquin --> Palestrina[팔레스트리나]

  Monteverdi[몬테베르디] --> Lully[륄리]
  Lully --> Handel[헨델]
  Handel --> Mozart[모차르트]
  Mozart --> Verdi[베르디]
  Verdi --> Wagner[바그너]

  Bach[바흐] --> Mozart
  Mozart --> Beethoven[베토벤]
  Beethoven --> Chopin[쇼팽]
  Chopin --> Liszt[리스트]
  Liszt --> Clara[클라라 슈만]
  Clara --> Debussy[드뷔시]

  Beethoven --> Brahms[브람스]
  Brahms --> Schoenberg[쇤베르크]
  Schoenberg --> Messiaen[메시앙]
  Messiaen --> Boulez[불레즈]

  Debussy --> Messiaen
  Wagner --> Mahler[말러]
  Mahler --> Schoenberg
  Nadia[나디아 불랑제] --> Stravinsky[스트라빈스키]
  Nadia --> Boulez

  Leipzig[라이프치히] --> Bach
  Vienna[빈] --> Mozart
  Vienna --> Beethoven
  Vienna --> Brahms
  Paris[파리] --> Chopin
  Paris --> Debussy
  Paris --> Nadia
  Bayreuth[바이로이트] --> Wagner
  Venice[베네치아] --> Monteverdi
  Rome[로마] --> Palestrina
  Milan[밀라노] --> Verdi
```

## 노드 분류 기준

- 인물 노드: 작곡가, 연주자, 교육자, 이론가
- 도시 노드: 활동 허브, 제도 허브, 재발견 허브
- 관계선: 영향, 전승, 후원, 재발견

## 다음 확장

1. 관계선에 색을 넣어 영향, 전승, 후원, 재발견을 구분한다.
2. 도시 노드를 따로 묶어 지리 허브 다이어그램을 만든다.
3. 인물별로 직접 연결선 3개 이상만 남긴 축약판도 만든다.
