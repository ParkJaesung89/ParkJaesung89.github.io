---
title: Github Workflows의 지속적인 Queued 상태 확인
date: 2024-07-19 10:30:00 +09:00
categories: [Github, Actions]
tags: [Github, Actions]
---

블로그 글 수정하기 위해 git push 를 사용했다. 그리고 습관적으로 github actions 탭을 확인했는데...  
방금 실행된 Workflow가 Queued 상태이다. 전달까지 아무문제 없었는데 갑자기 왜 이런 상태인 걸가?  
![Github_actions_queued](../assets/img/posts_img/Github_Actions_Queued/github_actions_queued.png)

해당 workflow를 삭제를 시도해도 삭제되지 않고 build 되어야 될 동작이 아예 시작조차 되지 않고 있다.  
![No_action_workflow](../assets/img/posts_img/Github_Actions_Queued/workflow%20동작%20안함.png)

[github의 상태 체크]  
workflows 파일이 변경된 것도 아니라 해당 파일 문제는 아닌 것 같다고 생각되며, 구글링 해보니 ubuntu 사용시에 "runs-on" 값이 "ubuntu-latest" 로 변경해야 된다는 글들이 확인되었다.
workflow 파일 확인해보니 해당 값도 "ubuntu-latest" 이다.
좀더 찾아본 결과 Github 서비스 자체의 문제일 수 있다고 한다. github 서비스에 대한 상태를 확인 가능한 사이트를 알게 되었다. 아래 경로로 가면 github의 상태를 확인할 수 있다.

[https://www.githubstatus.com](https://www.githubstatus.com/)

해당 사이트를 통해 현재 github의 이슈가 있는지 확인해보니 아래와 같이 "actions" 와 "Codespaces"가 Degraded 상태인 것을 확인 할 수 있으며, 위에 issue들에 대해서 현 상태를 실시간으로 업데이트 하고있다.

![github_status_check](../assets/img/posts_img/Github_Actions_Queued/github_status_check.png)

언제 정상화 되는 걸까...?
시간이 흘러서 아래와 같이 actions status가 Normal이 되었다.

![github_actions_recovery](../assets/img/posts_img/Github_Actions_Queued/github_actions_recovery.png)

github actions에서 workflow 확인하면 아직 이전에 있었던 Queued들은 그대로 이다. 변경된 부분 적용을 위해 다시 push하면 이전 push했던 것들이 제대로 처리되지 못하여 충돌이 난다.
아래와 같이 적용하여 해결한다.

```bash
sudo git add -A
sudo git commit -m"."
sudo git rebase --confinue
sudo git push origin
```

_Rebase는_ 한 브랜치의 커밋들이 다른 브랜치의 맨 끝으로 옮기는 Git 명령으로 git 히스토리를 깔끔하게 정리하여 충돌을 없애고 통합한다.

자, Github의 Actions을 다시 확인 해보자.
![workflow_success](../assets/img/posts_img/Github_Actions_Queued/workflow%20동작함.png)

정상적으로 동작 한 것을 확인 할 수 있다.  
이렇게, 설정의 문제가 아닌 github 자체의 문제일 경우도 있기 때문에 github의 status도 잘 확인하는 것이 좋다.
