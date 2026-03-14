# Relationship Diagram Cities

## 목적

- 도시를 중심 노드로 두고 어떤 인물과 제도가 집중되는지 빠르게 파악한다.

## Mermaid 초안

```mermaid
graph LR
  Vienna[빈] --> Haydn[하이든]
  Vienna --> Mozart[모차르트]
  Vienna --> Beethoven[베토벤]
  Vienna --> Brahms[브람스]
  Vienna --> Mahler[말러]
  Vienna --> Schoenberg[쇤베르크]

  Paris[파리] --> Chopin[쇼팽]
  Paris --> Liszt[리스트]
  Paris --> Debussy[드뷔시]
  Paris --> Nadia[나디아 불랑제]
  Paris --> Boulez[불레즈]
  Paris --> Stravinsky[스트라빈스키]

  Leipzig[라이프치히] --> Bach[바흐]
  Leipzig --> Clara[클라라 슈만]

  Rome[로마] --> Palestrina[팔레스트리나]
  Rome --> Josquin[조스캥]

  Venice[베네치아] --> Monteverdi[몬테베르디]
  Milan[밀라노] --> Verdi[베르디]
  Bayreuth[바이로이트] --> Wagner[바그너]
```

## 사용 메모

- 한 도시의 인물군을 읽을 때 `cities/*.md`와 `people/*.md`를 함께 연다.
- 도시 간 이동이 중요한 인물은 두 도시에 중복 연결해도 된다.
