## 1. Gitlab 업그레이드
- 현재 사내에서는 Docker와 EC2, AWS ELB를 활용하여 Gitlab을 사용중이다.
- Gitlab을 업그레이드 하기 위하여 살펴 보았는데 몇가지 문제가 존재하였다.
  - 서버 OS의 노후화
  - Gitlab 업그레이드 진행 도중 downtime(중단 시간) 발생
  - 실제 운영 Gitlab을 업그레이드시 리스크 존재
- 위 문제 모두 Gitlab을 특정 시간동안 이용할 수 없다는 문제가 있고, 그 시간이 몇 분이 아닌 적어도 30분 이상은 소요된다는 문제가 있다.
  - 가장 큰 문제는 현재 Jenkins를 이용하여 배치 작업을 진행중인데 Gitlab을 업그레이드 하는 동안에는 배치 운영이 불가하다.
- 위 문제를 해결하기 위해서 Gitlab 업그레이드 시 중단 시간을 최소화할 필요성을 느껴 이 글을 작성하게 되었다.
 
## 2. Gitlab 업그레이드 중단 시간 최소화 하기
- 기본적인 계획은 개발자가 Gitlab을 이용하지 않는 시간대에 백업을 하여, 새로운 서버에 업로드 및 복구 작업을 진행한다.
- 이후 AWS ELB의 대상 그룹을 기존 서버에서 새로운 서버로 변경하여 Gitlab의 중단 시간을 최소화 한다.
  - 마치 무중단 배포를 수동으로 한다고 생각하면 된다.

### 2.1. Gitlab 업그레이드 계획

#### 2.1.1 Gitlab을 어느 버전까지 업그레이드할지 정한다.
- Gitlab을 특정 버전대 별로 순차적으로 버전 업그레이드를 진행해야한다.
- 예를 들어서 16.11.10 -> 17.4.1 버전으로 업그레이드 하기 위해서는 16.11.10 버전을 17.3.4 버전 업그레이드 이후에 17.4.1 버전으로 업그레이드가 가능하다.
- Gtlab 업그레이드 경로는 [Gitlab > Upgrade Path Tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/) 링크를 통해서 확인이 가능하다.

#### 2.1.2 기존 서버에 존재하는 Gitlab을 백업한 이후 새로운 서버에 업로드 한다.
- 기존 서버에 존재하는 Gitlab 데이터를 백업한다.
```sh
## 백업 명령어
## Docker에 의해 마운트된 Volume에서 확인이 가능하며, 원본 경로는 /var/opt/gitlab/backups 이다.
docker exec -it <container name> gitlab-backup create
```
- 백업된 데이터는 sftp 명령어를 이용하여 새로운 서버에 업로드한다.

> [Gitlab > Backup](https://docs.gitlab.com/ee/install/docker/backup_restore.html)

#### 2.1.3 새로운 서버에 Gitlab 설치를 하는데, 기존 서버의 Gitlab과 동일한 버전을 설치한 이후 복구한다.

- 새로운 서버에 기존 Gitlab 서버와 동일한 버전의 Gitlab을 설치한다.
  - 위 작업을 진행하는 이유는 백업을 복구 시킬때, 기존 환경과 일치해야하기 때문에 기존 버전과 동일한 버전을 설치한다.
  ```yaml
  services:
  gitlab:
    image: gitlab/gitlab-ee:16.8.10-ee.0
    restart: always
    hostname: 'gitlab.example.com'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        gitlab_rails['gitlab_shell_ssh_port'] = 2424
    ports:
      - '4000:80'
      - '443:443'
      - '2424:22'
    volumes:
      - './gitlab/config:/etc/gitlab'
      - './gitlab/logs:/var/log/gitlab'
      - './gitlab/data:/var/opt/gitlab'
  ```
- 업로드된 백업 파일을 `/var/opt/gitlab/backups/`에 이동시킨 이후에, 권한을 부여한다.
```sh
sudo cp 11493107454_2018_04_25_10.6.4-ce_gitlab_backup.tar /var/opt/gitlab/backups/
docker exec -it <container name> chown git:root /var/opt/gitlab/backups/11493107454_2018_04_25_10.6.4-ce_gitlab_backup.tar
```
- Gitlab에서 사용중인 일부 프로세스를 중지시킨 이후에 상태 확인을 한다.
```sh
docker exec -it <container name> gitlab-ctl stop puma
docker exec -it <container name> gitlab-ctl stop sidekiq

docker exec -it <container name> gitlab-ctl status 
```
- 백업된 파일을 복구 시킨다.
```sh
### 백업 파일명에서 _gitlab_backup.tar는 생략한다.
docker exec -it <container name> gitlab-backup restore BACKUP=11493107454_2018_04_25_10.6.4-ce
```
- Gitlab 재시작 이후에 상태를 확인한다.
```sh
docker exec -it <container name> gitlab-ctl restart
docker exec -it <container name> gitlab-ctl status
docker exec -it <name of container> gitlab-rake gitlab:check SANITIZE=true
```

> [Gitlab > Restore](https://docs.gitlab.com/ee/administration/backup_restore/restore_gitlab.html#certain-gitlab-configuration-must-match-the-original-backed-up-environment)

#### 2.1.4. 새로운 서버의 Gitlab을 업그레이드한다. 
- `docker-compose.yml`파일의 이미지를 수정하고 저장한뒤 해당 명령어를 수행한다.
```sh
docker compose pull
docker compose up -d
```

> [Gitlab > Upgrade](https://docs.gitlab.com/ee/install/docker/upgrade.html)

#### 2.1.5. 업그레이드를 진행한 이후에 테스트를 진행한 이후 AWS ELB 대상 그룹을 변경한다.

- 테스트 리스트
  - 로그인 및 유저 리스트 확인
  - Git 프로젝트 확인
    - 리스트
    - 상세보기
  - issue와 merge request 접근 가능 여부 확인
  - clone, pull 및 push 가능 여부 확인
- 테스트가 완료된 이후에는 AWS ELB 대상 그룹을 변경하여 중단 시간을 최소화 한다.

> [Gitlab > post-upgrade checks](https://docs.gitlab.com/ee/update/plan_your_upgrade.html#pre-upgrade-and-post-upgrade-checks)

<br/> 
<br/> 

> [Gitlab > Upgrade Path Tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/) <br/> 
> [Gitlab > Backup](https://docs.gitlab.com/ee/install/docker/backup_restore.html) <br/> 
> [Gitlab > Restore](https://docs.gitlab.com/ee/administration/backup_restore/restore_gitlab.html#certain-gitlab-configuration-must-match-the-original-backed-up-environment)  <br/> 
> [Gitlab > Upgrade](https://docs.gitlab.com/ee/install/docker/upgrade.html)  <br/> 
> [Gitlab > post-upgrade checks](https://docs.gitlab.com/ee/update/plan_your_upgrade.html#pre-upgrade-and-post-upgrade-checks)  <br/> 