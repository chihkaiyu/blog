---
title: "Walkthrough Gitlab"
slug: "walkthrough-gitlab"
date: 2018-06-29T20:03:20+08:00
draft: false
image: "/content/images/2018/06/DSC03483.jpg"
tags: ["gitlab"]
---

這份文件會告讓你怎麼設定一個 GitLab project 以及到哪裡找到相關的官方文件。這份文件的順序是按照 GitLab project 左側的 sidebar撰寫，如果有項目不在你的 project 裡，有可能是你沒有權限或是沒有開啟該功能。
此份文件是根據 GitLab Community Edition 10.6.4 [dee2c87](https://gitlab.com/gitlab-org/gitlab-ce/commits/dee2c87) 所撰寫，同時亦有許多遺漏之處，不明確之處請查閱[官方文件](https://docs.gitlab.com/ce/README.html)。


# Overview
## Details
![Overview/Details](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1530267630997_details.png)


當你鍵入 project 的 URL 便會到此頁面。此頁內容可以在你的個人設定裡變更，可選擇的選項有：files and readme (default)、activity、readme。設定的位置在：`User Settings → Preferences → Behavior → Project overview content`。

## Activity
![Overview/Activity](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527765654703_activity.png)


按照時間顯示此 project 最近的活動，圖示最上方的紅框裡，可選擇要查看不同類型的活動。

## Cycle Analytics
![Overview/Cycle Analytics](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527765824619_cycle_analytics.png)


這頁顯示 project 的 release cycle，他替你量測從點子到上線需要花多少時間。需要注意的是，你需要在環境裡定義 `production` 或是 `production/*` 才會有用，否則系統無法蒐集到相關數據。目前我們甚少使用這個功能，你可能需要自己閱讀相關文件。
https://docs.gitlab.com/ce/user/project/cycle_analytics.html

# Repository
## Files
![Repository/Files](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527765798264_files.png)


此頁面顯示程式碼的資料夾結構，底下按照圖示上的數字順序解釋各按鈕功能：

1. 透過此下拉式選單切換 branch 或是 tag，亦可在此搜尋 branch 或 tag
2. 目前你所在的路徑
3. 點擊 `+` 可新增檔案、上傳檔案、新增資料夾、新增 branch、新增 tag
4. 點擊 `History` 可檢視目前路徑下的歷史紀錄，先點擊檔案再點擊 `History`，便是檢查單一檔案的歷史紀錄
5. 下載 source code，有各種壓縮格式可選擇
## Commits
![Repository/Commits](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527859847630_commits.png)


此頁面按照時間顯示該 project 的 commits，底下按照圖示上的數字順序解釋各按鈕功能：

1. 透過此下拉式選單切換 branch 或是 tag，亦可在此搜尋 branch 或 tag
2. 在此輸入文字過濾 commit message
## Branches
![Repository/Branches](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527860530362_branches.png)


此頁面顯示該 project 各 branch，上面紅框處可按照狀態過濾 branch，但目前找不到 stale branch 的意思，底下按照圖示上的數字順序解釋各按鈕功能：

1. `default` 表示預設 branch，在進入 project 會顯示預設 branch，任何分支操作如：發出 MR、compare 皆以預設 branch 為目標，可在 `Settings → General → General project settings → Default Branch` 更改
2. `protected` 表示此 branch 是受保護的，其受保護的定義請見下方 `Settings/Repository`
3. 顯示與預設 branch 的 commits 數差異，左邊為落後，右邊為超前
4. 發出 MR，來源 branch 為此 branch，目標 branch 為預設 branch
5. 與預設 branch 作比較
6. 下載此 branch 的 source code，有多種壓縮格式可選擇
7. `merged` 表示此 branch 已 merge 到預設 branch
8. 輸入文字過濾 branch
9. 刪除所有已 merged 且未受保護的 branch
## Tags
![Repository/Tags](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527861409492_tags.png)


顯示此 project 的 tag，底下按照圖示上的數字順序解釋各按鈕功能：

1. 輸入文字過濾 tag 名稱
2. tag 排序規則，可按照：由新到舊、由舊到新、tag 名稱
3. 編輯該 tag 的 release notes
## Contributors
![Repository/Contributors](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527861600701_contributors.png)


此頁面顯示 contributors 的 commits 數量，只計算預設 branch 的數量，底下按照圖示上的數字順序解釋各按鈕功能：

1. 透過此下拉式選單切換 branch 或是 tag，亦可在此搜尋 branch 或 tag
## Graph
![Repository/Graph](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527862001527_graph.png)


此頁面顯示該 project 的 git 路徑，底下按照圖示上的數字順序解釋各按鈕功能：

1. 可在此處輸入 commit sha 再按下右方按鈕可快速找出該 commit
2. 若勾選此處，則圖示最多只顯示到你輸入的 commit sha
3. 透過此下拉式選單切換 branch 或是 tag，亦可在此搜尋 branch 或 tag
4. 每一個點都是一個 commit，可點選跳至該 commit 看相關訊息
## Compare
![Repository/Compare](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1530330340364_Inkedcompare_LI.jpg)


此頁面可比對任兩個 commit 之間的差異，底下按照圖示上的數字順序解釋各按鈕功能：

1. 選擇來源 commit，點選此處可選擇 branch、tag 或是直接輸入 commit sha 亦可。此處 commit 需比 target 新
2. 選擇目前 commit，點選此處可選擇 branch、tag 或是直接輸入 commit sha 亦可。此處 commit 需比 source 舊
3. 將 source 與 target 對調
4. 選擇 diff 的呈現方式
5. 是否將空白列入 diff
6. 點選此處可直接複製該 commit 的 commit sha
7. 點選此處可直接前往該 commit
## Charts
![Repository/Chats](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527867675710_charts.png)


此頁面顯示該 project 的分析，上面為所含的程式語言比例，底下為 commit 的分析，分別統計 day of month, weekday, per day hour (UTC)，底下按照圖示上的數字順序解釋各按鈕功能：

1. 透過此下拉式選單切換 branch 或是 tag，亦可在此搜尋 branch 或 tag
# Issues
## List
![Issues/List](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1527868401316_list.png)


此頁面列出所有 issue，圖示上方紅框處可根據狀態切換 issue 列表，底下按照圖示上的數字順序解釋各按鈕功能：

1. 點選此處可顯示近期搜尋紀錄
2. 在此處輸入條件過濾 issue，點選後會有一下拉式選單給你以條件過濾，可以根據下列資訊過濾 issue：author、assignee、milestone、label、my-reaction (emoji)
3. 點選此處後，可在左方 issue 列表處勾選多個 issues，並在右方設定要更新的內容做批次更新 issues
4. Issues 排序依據，可以根據：priority、created date、late updated、milestone、due date、popularity、last popularity
5. 該 issue 所擁有的 label，點選 label 可直接過濾出所有擁有該 label 的 issue
## Board
![Issues/Board](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528040736119_board.png)


此頁面以 board 形式列出 issue，可根據需求以不同 label 建立不同 list，各 issue 會根據擁有的 label 歸類在不同 list，底下按照圖示上的數字順序解釋各按鈕功能：

1. 在此處輸入條件過濾 issue，點選後會有一下拉式選單給你以條件過濾，可以根據下列資訊過濾 issue：author、assignee、milestone、label、my-reaction (emoji)
2. 可將 issue 在 list 之間拖拉，會根據不同 list 在 issue 上增減相對應 label
3. 點擊按鈕在該 list 新增 issue
4. 點擊按鈕並選擇 label 新增一 list
## Labels
![Issues/Labels](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528091423064_labels.png)


此頁面顯示該 project 的 labels，上方紅框處是使用者自行標 star 的 labels，底下按照圖示上的數字順序解釋各按鈕功能：

1. 點擊此星星將該 label 標為重要 label，會移至圖示紅框處置頂
2. 點擊此連結會將你導至過濾出含有該 label 的 issues 頁面
3. 點擊此連結會將你導至過濾出含有該 label 的 merge request 頁面
4. 點擊此按鈕將此 label 升級為 group level label
5. 點擊此按鈕將編輯該 label，可編輯：title、description、background color
6. 點擊此按鈕刪除該 label
7. 訂閱該 label，當有任何 issue 或 merge request 被標上該 label 時，便會通知你
8. 可藉由拖拉 label 調整排序
9. 點擊 label 圖示會將你導至過濾出含有該 label 的 issues 頁面
## Milestones
![Issues/Milestones](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528092065463_milestones.png)


此頁面顯示該 project 的 milestones，milestones 可作為 release 版本的依據，將下次 release 相關 issues、merge requests 放進來一起管理。上方紅框處可依照狀態切換 milestones，底下按照圖示上的數字順序解釋各按鈕功能：

1. 點擊右方 `x Issues` 連結會將你導至過濾出含有該 milestone 的 issues 頁面
2. 點擊紅方 `x Merge Requests` 連結會將你導至過濾出含有該 milestone 的 merge requests 頁面
3. 點擊標題連結會將你導至該 milestone 的詳細頁面
4. Milestones 排序條件，可根據：due soon、due later、start soon、start later、name, ascending、name, descending
5. 擊點此按鈕可編輯該 milestone
6. 將此 milestone 升級為 group level milestone
# Merge Requests
![Merge Requests](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528092760940_merge_requests.png)


此頁面顯示該 project 的 merge requests，圖示上方紅框處可根據狀態切換 merge requests，底下按照圖示上的數字順序解釋各按鈕功能：

1. 點選此處可顯示近期搜尋紀錄
2. 在此處輸入條件過濾 merge requests，點選後會有一下拉式選單給你以條件過濾，可以根據下列資訊過濾 merge requests：author、assignee、milestone、label、my-reaction (emoji)
3. 點選此處後，可在左方 merge requests 列表處勾選多個 merge requests，並在右方設定要更新的內容做批次更新 merge requests
4. Merge requests 排序依據，可以根據：priority、created date、late updated、milestone、due date、popularity、last popularity
5. 顯示該 merge request 的 target branch
6. 點擊標題連結可至細詳頁面
# CI/CD
## Pipelines
![CI / CD/Pipelines](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528102032775_pipelines.png)


此頁面顯示該 project pipelines，若看不到此頁面，可能是你權限不夠，或是該 project 沒有開啟 pipelien 功能。圖示上方紅框處可根據狀態 pipelines，底下按照圖示上的數字順序解釋各按鈕功能：

1. 點擊此按鈕可看到整個 pipeline 各 stage 的完整 job 列表，同時亦是該 pipeline 的狀態，
2. 點擊此按鈕可展開該 stage 的所有 jobs，每個 job 旁邊會有一個 retry 的按鈕，點擊該按鈕便該單獨重跑該 job
3. Web 手動觸發 pipeline，可選擇要跑哪個 branch 或是 tag。在 `.gitlab-ci.yml` 可以設定某些 job 只在 web 手動觸發才跑，相關設定請看[官方文件](https://docs.gitlab.com/ce/ci/yaml/#only-and-except-simplified)
4. 清除該 project 在 GitLab Runner 上的 cache，cache 用法請看[官方文件](https://docs.gitlab.com/ce/ci/yaml/#cache)
5. 點擊此按鈕會導至另一個頁面，可輸入你的 `.gitlab-ci.yml` 驗證是否合法
6. 重跑整個 pipeline
## Jobs
![](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528102568292_jobs.png)


此頁面會以 job 為單位顯示出所有 job，圖示上方紅框處可根據狀態切換 job，底下按照圖示上的數字順序解釋各按鈕功能：

1. 點擊此按鈕可看到該 job 的 log output 及其他詳細資訊，同時亦是該 job 的狀態
2. 點擊此按鈕可重跑該 job
3. 該 job 的 code coverage，若在 `.gitlab-ci.yml` 有設定 coverage，便會依照設定的 regular expression 去 parse output，coverage 用法請看[官方文件](https://docs.gitlab.com/ce/ci/yaml/#coverage)
## Schedules

此頁面是讓你設定排程，可定時去 trigger pipeline。目前我們公司將此類 job 集中在 [Jenkins](http://jenkins.local.bridgewell.com) 上跑。其詳細用法請見[官方文件](https://docs.gitlab.com/ce/user/project/pipelines/schedules.html)。

## Environments

若你的 GitLab CI 有設定環境，便可在此頁面查看各環境的狀態，目前並沒有使用，請自行查閱[官方文件](https://gitlab.com/help/ci/environments)及 `[.gitlab-ci.yml](https://docs.gitlab.com/ce/ci/yaml/#environment)` [設定](https://docs.gitlab.com/ce/ci/yaml/#environment)。

## Kubernetes

公司目前有意朝 [Kubernetes](https://kubernetes.io/) 方向發展，但還在調查階段，目前 GitLab 亦沒有與 Kubernetes 整合，有興趣者可自行查看[官方文件](https://gitlab.com/help/user/project/clusters/index)。

## Charts
![CI / CD/Charts](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528103391179_charts.png)


此頁面顯示該 project 的 pipelines 統計狀況，可能可以作為該 project 的開發流程指標參考。此頁面沒有可以設定的東西，可自行查看各圖表意義。

# Wiki
![Wiki](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528103856530_wiki.png)


此頁面為該 project 的 Wiki 頁，可作為該 project 的 wikipedia，若看不到有可能是你沒有權限或是該 project 沒有開啟 wiki 功能。
詳細設定方法請查看[官方文件](https://gitlab.com/help/user/project/wiki/index.md)，或是公司已有 wiki 頁的 [project](https://gitlab.com/infrastructure/ansible/ansible_hdfs_deployment/wikis/home)。

# Snippets
![](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528103901527_snippets.png)


此頁面顯示該 project 的 snippets，snippets 是放一小片段的程式碼的地方，沒有版本控制功能，可為 snippets 設定 visibility，如圖示上方紅框處可根據 visibility 切換 snippets。

# Settings
## General
![Settings/General](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528104109004_general.png)


此設定頁面大多設定該 project 的基本行為，下列分項說明：
**General project settings**

- Project name：project 的名稱，可在此變更，不會改變 project 的 path
- Project description：形容該 project 的描述文字，會顯示在 project list 底下
- Default Branch：預設 branch，可在此變更，在 Repository/Branches 提到用法
- Tags：可替該 project 增加 tags，以逗號分隔，目前不清楚用途
- Project avatar：可替該 project 新增圖示或刪除圖示

**Permissions**

- Project visibility：可調整該 project 可以被誰看見，有 public、internal、private，詳細說明請見[官方文件](https://gitlab.com/help/public_access/public_access)
- 底下各功能可選擇開啟與否，選擇後可調整權限設定

**Merge request settings**

- Merge method：可調整 merge 方式，例如 merge 時是否產生 merge commit
- 底下各選項有各種 merge request 限制，例如 pipeline 一定要成功才能 merge、要等所有 discussions 被解決才能 merge…等等

**Export project**

- 可在此匯出整個 project
  - 下列資料會被匯出：
    - Project and wiki repositories
    - Project uploads
    - Project configuration including web hooks and services
    - Issues with comments, merge requests with diffs and comments, labels, milestones, snippets, and other project entities
  - 下列資料**不**會被匯出：
    - Job traces and artifacts
    - LFS objects
    - Container registry images
    - CI variables
    - Any encrypted tokens

**Advanced settings**

- Housekeeping：會清理該 project，例如壓縮 file revisions 及刪除已不存在的物件，實際指令是跑 `git gc` 及 `git repack`
- Archive project：封存該 project，封存後該 project 變為唯讀，並且不會出現在 dashboard 亦不會出現在搜尋列。封存後的 project 也不能再被 commit
- Rename repository
  - Project name：變更該 project 的名稱，並不影響路徑，只影響搜尋時的判別
  - Path：變更該 project 的路徑，此行為會影響範圍甚廣，請小心使用。變更後你需要變更 local 的 git remote
- Transfer project：將該 project 轉到另一個 namespace 底下，影響甚廣請小心使用。只能轉到由你管理的 namespace 底下，需要變更 local 的 git remote，project visibility 會變為與新 namespace 一致
- Remove project：刪除整個 project 及所有相關的 issues、merge requests 等等，刪除無法復原，請小心使用，建議一律使用封存即可
## Members
![](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528109074255_members.png)


此頁面顯示該 project 的 members 狀態及其權限，若要新增/移除 members 權限亦在此頁面。底下按照圖示上的數字順序解釋各按鈕功能：

![Settings/Members/Share with group](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528108639391_share_group.png)

1. 如上圖所示，可將該 project 直接分享給其他 group，並可設定最高權限為何者以及過期日期，最後按下 10. 底下的按鈕即可
2. 選擇要邀請哪個 user 到此 project
3. 選擇該 user 的身份，其身份與權限對照表請查看[官方文件](https://docs.gitlab.com/ce/user/permissions.html)
4. 選擇該 user 在此 project 的身份何時過期，若不填則為永久
5. 輸入文字過濾 users
6. Users 排序規則
7. 若此處有按鈕，代表該 user 的權限是你邀請的，並不是從 parent group 繼承來的，你可以編輯該 user 的權限，此處為調整該 user 身份
8. 此處為調整該 user 期限
9. 刪除該 user 在此 project 的身份
10. 如 1. 所說明
11. 若此處有連結，代表該 user 在此 project 的權限是由這個連結的 group 繼承來的
## Integrations
![Settings/Integrations](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528256416770_integrations-1.png)


此頁面顯示及讓你設定該 project 與外部的 integration，底下按照圖示上的數字順序解釋各按鈕功能：

1. 當有事件觸發時，要將 HTTP request 送去何處，在此填入該 server 的 URL
2. 可自訂 token 來驗證來源是否真正，此 token 會放在 `X-Gitlab-Token` HTTP header 裡
3. 選擇哪些事件要觸發 webhook，各事件詳細敘述請見[官方文件](https://docs.gitlab.com/ce/user/project/integrations/webhooks.html)
4. 選擇是否驗證 SSL，公司 GitLab 架在內網，目前並未設定 SSL，請取消勾選
5. 上述設定完成後點擊此按鈕加入
6. 已加入的 webhoook，可以加入多個，底下會顯示該 webhook 所負責的事件
7. 點擊此按鈕編輯 webhook
8. 點擊此按鈕測試 webhook，點擊後會有一下拉式選單選擇要測試哪個事件，測試資料會與真實有所差異，在測試時請注意
9. 刪除該 webhook
![](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528256907101_integrations-2.png)


除了自行設定 webhook 外，GitLab 亦內建許多與其他 service 整合好的設定，點選各 service 名稱即可進入設定，由於各 service 迥異在此不一一詳述，請參考官方文件：

- [GitLab Integration](https://docs.gitlab.com/ce/integration/)
- [Project Services](https://docs.gitlab.com/ce/user/project/integrations/project_services.html)
## Repository
![Settings/Repository](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528257253566_repository.png)


此頁面設定該 project 受保護的 branches 及 tags，防止有人不小心 push branch 或 tag 直接觸發 pipeline 或覆蓋掉，下列分項說明：

**Protected Branches**

- Protected branches：用來保護特定 branches 穩定，強迫開發者一定要發 merge request 而不是直接 push，protected branches 有下列特性
  - prevent their creation, if not already created, from everybody except Masters
  - prevent pushes from everybody except Masters
  - prevent **anyone** from force pushing to the branch
  - prevent **anyone** from deleting the branch
- Protect a branch
  - Branch：選擇要保護的 branch name，可下 wildcard，例如 `*-stable` 或 `production/*`
  - Allowed to merge：選擇何種身份的人可以 merge 該 banch
  - Allowed to push：選擇何種身份的人可以 push 該 branch
- Protected branch：已受保護的 branch，可在此更改設定或是 unprotect
- 詳細資訊請參考官方文件：[protected branches](https://docs.gitlab.com/ce/user/project/protected_branches.html) 及 [permission](https://docs.gitlab.com/ce/user/permissions.html)

**Protected Tags**

- Protected tags：用來保護特定 tag，避免某些 tag 被 create 或 update，protected tags 有下列特性
  - prevent their creation, if not already created, from everybody except Masters
  - prevent pushes from everybody except Masters
  - prevent **anyone** from force pushing to the branch
  - prevent **anyone** from deleting the branch
- Protect a tag
  - Tag：選擇要保護的 tag name，可下 wildcard，例如 `v*` 或 `*-release`
  - Allowed to create：選擇何種身份的人可以 create 該 tag

**Deploy Keys**
根據官方說明：若有其他系統要與 GitLab 整合，可以使用 deploy keys，便不需要建立一個 dummy user account，詳細說明請參考[官方文件](https://docs.gitlab.com/ce/ssh/#deploy-keys)

## CI / CD
![Settings/CI / CD](https://d2mxuefqeaa7sj.cloudfront.net/s_5E4B35A94B1491785967114A2E7FA68DF3BE2DE2CA1343D25FAADB12CF6B883F_1528258468039_CI_CD.png)


此頁面讓你設定有關該 project 的 CI / CD 事項，下列分項說明：
**General pipelines settings**

- Auto DevOps (Beta)：GitLab 的 feature 之一，其賣點如同他的名稱，但需要架設 Kubernetes，且還是 beta 版，公司目前沒有啟用，詳細說明請見[官方文件](https://docs.gitlab.com/ce/topics/autodevops/)
- Runner token：當你要設定某個 GitLab Runner 是特定只跑該 project 時要給的 token，不建議自行架設 GitLab Runner，請聯絡 GitLab Runner 負責人
- Git strategy for pipelines：CI/CD pipeline 時，要怎麼拿到 code base，下列說明 `clone` 與 `fetch` 的差異處：
  - `git clone`：which is slower since it clones the repository from scratch for every job, ensuring that the project workspace is always pristine.
  - `git fetch`：which is faster as it re-uses the project workspace (falling back to clone if it doesn't exist).
- Timeout：一個 job 跑多久就 timeout 並視為 fail，單位是分鐘，預設是 60 分鐘
- Custom CI config path：GitLab CI 的 config 路徑，預設是第一層的 `.gitlab-ci.yml`
- Public pipelines：是否公開 pipelines 的內容，包含 output logs，其設定會受該 project 的 visibility 影響
- Auto-cancel redundant, pending pipelines：是否自動取消較舊的 pipelines
- Test coverage parsing：輸入 regular expression 從 job output logs 解析出 coverage，亦可在 `.gitlab-ci.yml` 設定
- Pipeline status：可選擇不同 branches、tags 的 pipeline status badge，可放在 README 顯示
- Coverage report：可選擇不同 branches、tags 的 coverage report badge，可放在 README 顯示

**Runners settings**
查看 GitLab Runner 與該 project 的設定，可於此拿到該 project 的 token 設置一個專跑該 project pipeline 的 GitLab Runner，目前不建議這麼做，請聯絡 GitLab Runner 負責人。
右方 Shared Runners 處要注意是否 enable shared runner for this project，若關閉的話，該 project 的 pipeline 只能跑在特定為該 project 跑的 GitLab Runner，

**Secret variables**
Secret variables 用來藏密碼、secret keys 等較 sensitive 的資訊，在跑 CI/CD pipeline 時，GitLab Runner 會將這些 secret variables export 到環境變數，亦可在最右邊勾選 protected，只讓 protected branches 及 protected tags 可以取用這些 secret variables。詳細內容請見[官方文件](https://docs.gitlab.com/ce/ci/variables/README.html#secret-variables)。

**Pipeline triggers**
此處讓你設定 pipeline triggers，設定完之後可發送 HTTP request 來觸發 pipeline，詳細設定請見[官方文件](https://docs.gitlab.com/ce/ci/triggers/README.html)。


# References
- [GitLab Help Page](https://gitlab.com/help)
  GitLab instance 的 help 頁面，較為簡陋，大部份內容與官方文件上一樣
- [GitLab Docs](https://docs.gitlab.com/)
  GitLab 官方文件的入口，可在此搜尋，需注意的是，文件會有分 EE 版與 CE 版，我們的是 CE (community edition)，文件網址會顯示，可直接將 `ee` 改為 `ce`，例如：https://docs.gitlab.com/ee/README.html 可直接改為 https://docs.gitlab.com/ce/README.html 即是 CE 版本。

