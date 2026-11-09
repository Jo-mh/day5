# TheCatAPI 사용법 정리

조사일: 2026-11-09
출처: https://thecatapi.com/ , https://docs.thecatapi.com/ (reference, authorization, errors, examples), https://api.thecatapi.com/openapi-json

> 참고: `developers.thecatapi.com`은 `docs.thecatapi.com`으로 301 리다이렉트됩니다. 현재 API는 이미지, 품종, 투표, 즐겨찾기 외에 반려동물, 건강 팁, 웹훅 등도 포함합니다.
> 일부 내용(reference/OpenAPI 후반부)은 요약본에서 가져온 것이라 공식 문서로 재확인이 필요합니다.

## 1. 기본 정보
- Base URL: `https://api.thecatapi.com/v1`
- 인증 헤더: `x-api-key: YOUR-API-KEY`
- 키 발급: https://thecatapi.com/signup (이메일로 발급)

## 2. 인증과 요금제
- 문서에는 모든 요청에 `x-api-key` 헤더를 포함하라고 되어 있습니다.
- `GET /images/search`, `GET /images/{id}`는 OpenAPI 스펙상 공개 엔드포인트로 표시되어 있고, `mime_types` 설명에는 "키가 있든 없든 동작"이라고 되어 있습니다. 키 없이 쓸 때의 제한은 문서에 명시되어 있지 않으므로 직접 확인이 필요합니다.
- 키가 없거나 잘못되면 403이 반환됩니다.
- 요금제:

| 플랜 | 가격 | 월 요청 수 |
|---|---|---|
| Free | $0 | 10,000 |
| Startup | $99/월 | 100,000 |
| Enterprise | 문의 | 무제한 |

- Free 플랜은 분당 10회 요청으로 제한됩니다.
- 429 응답에 대한 `Retry-After`, `X-RateLimit-*` 헤더는 문서에 없습니다.

## 3. 랜덤 고양이 이미지 (핵심)

```
GET https://api.thecatapi.com/v1/images/search
```

```bash
curl -H "x-api-key: YOUR-API-KEY" "https://api.thecatapi.com/v1/images/search?limit=2"
```

응답 예시:

```json
[
  {
    "id": "KWdLHmOqc",
    "url": "https://cdn2.thecatapi.com/images/KWdLHmOqc.jpg",
    "width": 3114,
    "height": 2609,
    "breeds": []
  }
]
```

### 주요 쿼리 파라미터
| 파라미터 | 설명 |
|---|---|
| `limit` | 개수. 키가 있으면 1~100 |
| `page` | 0부터 시작하는 페이지 번호 |
| `order` | `ASC`, `DESC`, `RANDOM` (문서 일부에는 `RAND`로 표기) |
| `has_breeds` | 품종 정보가 있는 이미지만 |
| `breed_ids` | 쉼표로 구분한 품종 ID (예: `abys,beng`) |
| `include_breeds` | 품종 객체를 응답에 포함 |
| `include_categories` | 카테고리 객체를 응답에 포함 |
| `mime_types` | `jpg`, `png`, `gif` 등 (쉼표 구분) |
| `size` | `thumb`, `small`, `med`, `full`. 호환용이며 현재는 모든 값이 같은 URL을 반환 |
| `sub_id` | 해당 sub_id로 업로드한 이미지만 |

`category_ids`와 `format`은 스펙에 없습니다. 카테고리는 `GET /categories`를 사용합니다.

### 기타 이미지 엔드포인트
- `GET /images/{id}`: 이미지 한 장 조회
- `POST /images/upload`: 업로드 (`multipart/form-data`, 필드명 `file`)
- `GET /images/IMAGE_ID/status`: 업로드 처리 상태 (`clean`, `unsafe`, `labelled`)
- `GET /images?sub_id=...`: 내가 올린 이미지 목록
- `DELETE /images/IMAGE_ID`: 삭제
- 파일 크기 제한: 사진 50MB, 영상 200MB

## 4. 품종 (Breeds)
- `GET /breeds`: 전체 목록 (`limit` 기본 1000, `page`, `order`, `extinct` 등)
- `GET /breeds/search?q=siamese`: 이름 검색 (`q`는 1~30자, 별칭도 검색)
- `GET /breeds/{id}`: 상세 (예: `abys`, `beng`, `siam`)
- `GET /breeds/breed-groups`, `GET /breeds/{id}/colours`
- `lang` 파라미터(`it`, `de`, `es`, `fr` 등)는 Startup 이상에서 사용 가능합니다.
- Free 플랜은 핵심 필드(이름, 성격, 원산지, 수명, 설명, 이미지)만 제공됩니다. 문서마다 서술이 달라 확인이 필요합니다.

## 5. 투표 (Votes)
```bash
curl -X POST -H "Content-Type: application/json" -H "x-api-key: YOUR-API-KEY" \
  -d '{"image_id": "KWdLHmOqc", "value": 1}' \
  https://api.thecatapi.com/v1/votes
```
- `GET /votes`, `GET /votes/{vote_id}`
- 투표는 삭제할 수 없습니다. 되돌리려면 반대 `value`로 다시 투표합니다.

## 6. 즐겨찾기 (Favourites)
- `POST /favourites`: 본문 `{"image_id": "...", "sub_id": "..."}` (`sub_id`는 선택)
- `GET /favourites`, `GET /favourites/{id}`, `DELETE /favourites/{id}`

## 7. 페이지네이션
- 파라미터: `limit`, `page`(0부터), `order`
- 응답 헤더: `pagination-count`(전체 개수), `pagination-page`, `pagination-limit`
- `/facts`, `/health-tips`는 `page` 대신 `offset`을 사용합니다.

## 8. 에러 형식
```json
{
  "statusCode": 404,
  "timestamp": "2026-10-06T03:03:08.534Z",
  "path": "/v1/pets/doesnotexist",
  "message": "Pet with id \"doesnotexist\" not found",
  "error": "Not Found"
}
```
- 일부 에러에는 `code` 필드가 추가됩니다. `message`가 아니라 `statusCode`와 `code`로 분기합니다.
- 주요 상태 코드: 400(잘못된 요청 또는 쿼터 소진), 401, 403(키 없음/무효), 404, 429(요청 과다), 500, 503
- 재시도는 429, 500, 503만 하고 지수 백오프를 사용합니다. 나머지는 재시도해도 같은 결과입니다.

## 9. 기타 엔드포인트
`/facts`, `/health-tips`, `/pets`, `/webhooks`, `/accounts/quota`(`quota`, `usage`, `remaining`), `/accounts/features` 등이 있습니다. 자세한 내용은 https://docs.thecatapi.com/reference 를 참고하세요.

## 10. 이 프로젝트(index.html)에 적용할 때
```js
const res = await fetch("https://api.thecatapi.com/v1/images/search", {
  headers: { "x-api-key": "YOUR-API-KEY" } // 키 없이 쓸 수 있는지는 테스트 필요
});
const [cat] = await res.json();
img.src = cat.url;
```

**주의사항**
- CORS 정책과 브라우저 사용에 대한 언급은 문서에서 찾지 못했습니다. `x-api-key`는 커스텀 헤더라 프리플라이트 요청이 발생할 수 있으며, 이는 문서가 아닌 추정입니다.
- 브라우저 JS에 키를 넣으면 페이지를 여는 누구에게나 노출됩니다. 공개 배포 시에는 서버 프록시 사용을 권장합니다.
- Free 플랜은 분당 10회 제한이 있으므로 버튼 연타 시 429가 날 수 있습니다. 쿨다운이나 재시도 처리가 필요합니다.
