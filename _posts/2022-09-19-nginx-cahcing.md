---
title: "Nginx Caching"
layout: "post"
author: "skull(김윤기)"
header-style: text
comments: true
tags:
- nginx
- image caching
---

땡쿠 서비스는 정해진 프로필 사진 종류 내에서 원하는 프로필 이미지를 선택할 수 있습니다.
또 한 프로필 이미지 종류가 많지 않기에 WAS에 해당 이미지들을 저장해 놓고 필요시 이미지를 서빙하게 로직을 구성했었습니다.

하지만 이런 방식은 매번 이미지를 요청할 때마다 WAS에서 정적 파일을 응답해야 하기에 WAS에 불필요한 부하를 준다는 생각이 
들었고 이를 Nginx를 통해 개선하게 되었습니다.  

# Nginx를 어떤 용도로 사용할 것인가
Nginx로 위의 문제를 해결하기 위해 두가지 방식을 생각했었습니다.
- Nginx가 존재하는 EC2 인스턴스에 이미지 파일을 저장하고 Nginx가 이미지 파일을 응답
- Nginx를 캐싱으로 활용해 주어진 cahing 기간 동안에는 캐싱된 이미지를 Nginx에서 응답.
  Caching이 되지 않았다면 WAS에서 응답.  

이 방식들 중 caching을 하는 방식을 선택하게 되었습니다. 그 이유는 추후 프로필 이미지 종류 변경을 고려했기 떄문입니다.  
현재 was 코드 중 이미지 관련 로직 처리를 위해 이미지 파일 이름들을 가져오는 로직이 필요합니다. 현재 이 로직은 다음과 같습니다. 
```javascript
@Component
public class FileProfileImageGenerator implements ProfileImageGenerator {
    ...
    private static final int RANDOM_MIN_RANGE = 0;
    private final String imageUrlPath;
    private final List<String> profileImages;
    private final Random random;

    private FileProfileImageGenerator(@Value("${profile-image.image-url-path}") final String imageUrlPath,
                                      @Value("${profile-image.image-path}") final String imagePath) {
        this.random = ThreadLocalRandom.current();
        this.imageUrlPath = imageUrlPath;
        try {
            Resource[] resources = (new PathMatchingResourcePatternResolver())
                    .getResources(imagePath);
            profileImages = Arrays.stream(resources)
                    .map(Resource::getFilename)
                    .collect(Collectors.toList());
        } catch (IOException e) {
            throw new IllegalArgumentException(ErrorType.INVALID_PATH.getMessage());
        }
    }
    ...
}
```
이렇게 이미지 이름들을 was가 알아야 하는 상황에서 Nginx가 이미지 원본을 가지고 응답한다면 was는 nginx가 가지고 있는 이미지 이름들의 리스트를 알거나 같은 이미지들을 가지고 있어야 합니다.  
그러면 추후 프로필 이미지들에 변동 발생 시 관리 포인트가 늘어나게 되겠죠. 따라서 Nginx를 활용하되 caching을 사용하기로 했습니다.

# Nginx에 Caching 설정하기
Nginx에 적용한 caching 설정은 다음과 같습니다.
```yaml
http {

 ...
  # 캐시 저장소 설정
  # - keys_zone: 캐시 이름
  # - max_size: 전체 캐시 크기
 proxy_cache_path /data/nginx/cache/ keys_zone=profile-image:10m max_size=2024m use_temp_path=off;

  location  ~* \.(?:svg) {
    # 사용할 캐시 설정
    proxy_cache profile-image;
  
    # 캐싱할 메서드 종류
    proxy_cache_methods GET;

    # X-Proxy-Cache 헤더에 HIT, MISS, BYPASS와 같은 캐시 적중 상태정보가 설정
    add_header X-Proxy-Cache $upstream_cache_status;

    # 캐시 유효 기간 설정
    # proxy_cache_valid에 응답 코드를 명시하지 않으면 default로 200, 301, 302 응답을 캐싱
    proxy_cache_valid 2592000s;

    # Cache-Control을 public 으로 설정
    add_header Cache-Control "public";

    # 클라이언트의 헤더를 무시한다.
    # - “X-Accel-Expires”, “Expires”, “Cache-Control”, “Set-Cookie”, “Vary” 의 헤더는 캐시에 영향을 줄 수 있다.
    proxy_ignore_headers X-Accel-Expires Expires Cache-Control;

    # 만료기간을 1 달로 설정
    # Client cache 만료 기간
    #expires 1M;

    proxy_pass http://api;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $Host;
  }
}

```