---
title: Github 활용한 Jekyll thema 블로그 만들기(업데이트 중)
date: 2023-09-18 17:23:00 +/-0000
categories: [home]
tags: [hello world] # TAG names should always be lowercase
---

# 블로그 생성 과정(windows 환경)

## 윈도우즈 환경에서 jekyll을 설치(1. local환경 , 2. github repo)

1. github에 blog용 repo 생성
   repository name = {githubname}.github.io
   branch = main

   > github repo에 index.html 파일 생성 후 commit 하여, blog 링크 들어가면 생성한 페이지가 뜬다.

   - local 저장소에 clone
     ```bach
     git clone {github repo 경로}
     ```

2. Local 환경에 jekyll 설치  
   참고링크 : [윈도우즈 환경 jekyll 설치](https://wlqmffl0102.github.io/posts/Making-Git-blogs-for-beginners-2/)

   1. ruby 설치하며, 진행시 가장 마지막 단계에서 "ridk install" 진행  
      `ruby 3.1.x 버전으로 설치 할 것(현재는 최신버전도 문제 없는 것 같음)`
   2. CMD 창에서 jekyll과 bundler 설치
      ```bash
      gem install jekyll bundler
      jekyll -v
      ```
   3. CMD에서 local repo 폴더로 들어가서 명령어 실행
      ```bash
      cd {local repo 경로}
      chcp 65001        #인코딩 - 폴더 경로가 한글이 있을 경우 사용
      gem install jekyll bundler
      gem install webrick
      jekyll new ./
      bundle install
      bundle exec jekyll serve --trace
      ```
   4. 정상 설치 완료 시 "http://127.0.0.1:4000" 으로 접근 가능하다.
      > 만약 bundle exec jekyll serve 실행 시 아래와 같은 에러가 발생하면 포트 바인딩이 안되는 것으로 포트를 확보하거나 다른 포트로 변경하도록 한다.

   ```bash
   Permission denied - bind(2) for 127.0.0.1:4000 (Errno::EACCES)

   # 포트 변경하여 실행
   bundle exec jekyll serve --trace --port 5000
   ```

   - 로컬 pc에서 127.0.0.1:5000 접속 시에 페이지 뜨는 것 확인 가능

   5. chirpy 테마 적용방법  
      [chirpy-starter 링크](https://github.com/cotes2020/chirpy-starter)  
      위 링크 접속하여 코드 압축 폴더 다운로드 후 local repo에 전부 덮어쓰기

      - 이후에 \_config.yml 파일 수정해야 됨.(아래 설정 참고하여 작성)

   ```bash
   theme: jekyll-theme-chirpy  # import하는 테마 명입니다. 디폴트로 사용중인 테마 명이 들어가 있으므로, 수정은 하지 않습니다.

   baseurl: ''     # 사용자 페이지를 만들었을 경우, 빈칸으로 둡니다. 프로젝트 페이지를 만든 경우 프로젝트 명을 적어줍니다.
                  # 사용자 페이지와 프로젝트 페이지를 모르신다면, [초보자를 위한 GitHub Blog 만들기 - 1](https://wlqmffl0102.github.io/posts/Making-Git-blogs-for-beginners-1/)에서 Step 1-2. repository생성을 참고하시길 바랍니다.
   =
   lang: ko-KR     # 사용하는 언어 설정을 진행합니다. http://www.lingoes.net/en/translator/langcode.htm 로 접속하여 확인가능합니다.

   timezone: Asia/Seoul    #timezone설정입니다. http://www.timezoneconverter.com/cgi-bin/findzone/findzone 에서 확인가능합니다.

   # jekyll-seo-tag settings › https://github.com/jekyll/jekyll-seo-tag/blob/master/docs/usage.md
   # ↓ --------------------------

   title: SSinabro                          # 블로그 이름입니다. 설정하면 브라우저 상단에 설정된 이름이 확인가능합니다.

   tagline: 씬나부러   # 서브 타이틀 입니다. 설정하면 블로그 첫 페이지 좌측에서 확인 가능합니다.

   description: >-                        # "used by seo meta and the atom feed"라고 나옵니다. 저는 설정을 그대로 두었습니다.
   A minimal, portfolio, sidebar,
   bootstrap Jekyll theme with responsive web design
   and focuses on text presentation.

   url: "https://parkjaesung89.github.io/"    # 'https://username.github.io'와 같이 설정합니다. 설정이 잘 못 되면 곤란합니다.
                                          # 잘 적어넣도록 합니다.

   github:
   username: ParkJaesung89             # 본인의 github username을 적습니다.

   #twitter:
   #  username: twitter_username            # 본인의 twitter username을 적습니다. 저는 트위터는 사용하지 않아 주석 처리 해 두었습니다.

   social:
   # Change to your full name.
   # It will be displayed as the default author of the posts and the copyright owner in the Footer
   name: SSinabro
   email: batson1@naver.com             # change to your email address
   links:
      # The first element serves as the copyright owner's link
      #- https://twitter.com/username      # change to your twitter homepage
      - https://github.com/ParkJaesung89       # change to your github homepage
      # Uncomment below to add more social links
      # - https://www.facebook.com/username
      # - https://www.linkedin.com/in/username

      # 상단은 social관련 내용입니다. 본인의 이름, 이메일, 링크 등을 작성합니다. 저는 깃허브만 올려두었습니다.

   google_site_verification: '000' # Google Search Console관련 내용입니다.

   # ↑ --------------------------


   google_analytics:
   id: '000'              # Google Analytics ID입니다. 이 또한 이후 포스팅에서 다루겠습니다.
   pv:
      proxy_endpoint:
      cache_path:

   theme_mode:   # chirpy테마는 [light|dark]테마를 지원합니다. 비워두시면 사용자의 디폴트 값이 설정되고, light 또는 dark로 입력해두시면 페이지의 기본 테마가 설정됩니다.

   img_cdn: ''  #cdn 이미지 설정입니다. 저는 따로 진행하지 않았으나 진행하시려면 url을 작성해주시면 됩니다.

   avatar: /assets/img/profile.png     # 대표이미지 라고 생각하시면 됩니다. /assets/img경로에 사진을 넣은 뒤 작성하시면 됩니다.

   toc: true       # toc(Table of contents)입니다. 블로그 보시다 보면 포스팅 옆에서 스크롤을 따라오는 목차같은 녀석이 있습니다.
                  # 사용하시려면 true, 아니라면 false를 적으시면 됩니다.

   paginate: 10

   # ------------ 아래로는 크게 손 볼 것 없어서 생략합니다. ------------------
   ```

   6. 프로필에 쓸 이미지를 위에 설정과 같이 /assets/img/profile.png 에 넣어준 후 적용
      ```bash
      bundle exec jekyll serve --trace --port=5000
      ```
      > 127.0.0.1:5000에 접속하여 설정 변경된 것 확인

3. github 환경에 jekyll 올리기
   1. local에서 page가 잘 변경되었으면, github repo에 푸쉬한다.
      ```bash
      git add .
      git commit -m "Upload jekyll blog"
      git push
      ```
   2. git push가 정상적으로 된 것이 확인되면 github repo의 action 탭에 들어가서 build & deploy 확인
      - build 와 deploy에 에러가 발생하지 않으면 정상적으로 페이지에 적용 완료된다.
   3. 페이지 확인
      > deploy 완료 후 약 10분정도는 지나야 update 됨.

## 커스터 마이징

1. sidebar 이미지 넣기
   /\_sass/addon/commons.scss 파일에서 아래 설정으로 변경

```bash
# background: var(--sidebar-bg);        #기존에 존재하는 설정으로 이설정 삭제
background-image: url('/assets/img/sidebar-bg-img.jpg'); # 사용하고 싶은 이미지 경로
background-size: cover;  # 이미지의 size가 보여질 부분의 사이즈와 다른 경우에는 이미지 크기를 꽉차게 만든다
background-repeat: no-repeat; # 이미지의 size가 보여질 부분의 사이즈와 다른 경우 이미지가 반복하여 나오는데, 반복하지 않겠다.
```
