---
title:  "[Level1] 신고 결과 받기 for Kakao"

categories: algorithmQuestion

toc: true
toc_sticky: true

date: 2022-03-20
last_modified_at: 2022-03-20
---

# 신고 결과 받기

[신고 결과 받기](https://programmers.co.kr/learn/courses/30/lessons/92334)

## 문제

신입사원 무지는 게시판 불량 이용자를 신고하고 처리 결과를 메일로 발송하는 시스템을 개발하려 합니다. 무지가 개발하려는 시스템은 다음과 같습니다.

각 유저는 한 번에 한 명의 유저를 신고할 수 있습니다.
신고 횟수에 제한은 없습니다. 서로 다른 유저를 계속해서 신고할 수 있습니다.
한 유저를 여러 번 신고할 수도 있지만, 동일한 유저에 대한 신고 횟수는 1회로 처리됩니다.
k번 이상 신고된 유저는 게시판 이용이 정지되며, 해당 유저를 신고한 모든 유저에게 정지 사실을 메일로 발송합니다.
유저가 신고한 모든 내용을 취합하여 마지막에 한꺼번에 게시판 이용 정지를 시키면서 정지 메일을 발송합니다.
다음은 전체 유저 목록이 ["muzi", "frodo", "apeach", "neo"]이고, k = 2(즉, 2번 이상 신고당하면 이용 정지)인 경우의 예시입니다.

<table>
    <thead>
        <tr>
            <th>유저 ID</th>
            <th>유저가 신고한 ID</th>
            <th>설명</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>"muzi"</td>
            <td>"frodo"</td>
            <td>"muzi"가 "frodo"를 신고했습니다.</td>
        </tr>
        <tr>
            <td>"apeach"</td>
            <td>"frodo"</td>
            <td>"apeach"가 "frodo"를 신고했습니다.</td>
        </tr>
        <tr>
            <td>"frodo"</td>
            <td>"neo"</td>
            <td>"frodo"가 "neo"를 신고했습니다.</td>
        </tr>
        <tr>
            <td>"muzi"</td>
            <td>"neo"</td>
            <td>"muzi"가 "neo"를 신고했습니다.</td>
        </tr>
        <tr>
            <td>"apeach"</td>
            <td>"muzi"</td>
            <td>"apeach"가 "muzi"를 신고했습니다..</td>
        </tr>
    </tbody>
</table>

각 유저별로 신고당한 횟수는 다음과 같습니다.

<table>
    <thead>
        <tr>
            <th>유저 ID</th>
            <th>신고당한 횟수</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>"muzi"</td>
            <td>1</td>
        </tr>
        <tr>
            <td>"frodo"</td>
            <td>2</td>
        </tr>
        <tr>
            <td>"apeach"</td>
            <td>0</td>
        </tr>
        <tr>
            <td>"neo"</td>
            <td>2</td>
        </tr>
    </tbody>
</table>

	
위 예시에서는 2번 이상 신고당한 "frodo"와 "neo"의 게시판 이용이 정지됩니다. 이때, 각 유저별로 신고한 아이디와 정지된 아이디를 정리하면 다음과 같습니다.

<table>
    <thead>
        <tr>
            <th>유저 ID</th>
            <th>유저가 신고한 ID</th>
            <th>정지된 ID</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>"muzi"</td>
            <td>["frodo", "neo"]</td>
            <td>["frodo", "neo"]</td>
        </tr>
        <tr>
            <td>"frodo"</td>
            <td>["neo"]</td>
            <td>["neo"]</td>
        </tr>
        <tr>
            <td>"apeach"</td>
            <td>["muzi", "frodo"]</td>
            <td>["frodo"]</td>
        </tr>
        <tr>
            <td>"neo"</td>
            <td>없음</td>
            <td>없음</td>
        </tr>
    </tbody>
</table>
		
따라서 "muzi"는 처리 결과 메일을 2회, "frodo"와 "apeach"는 각각 처리 결과 메일을 1회 받게 됩니다.

이용자의 ID가 담긴 문자열 배열 id_list, 각 이용자가 신고한 이용자의 ID 정보가 담긴 문자열 배열 report, 정지 기준이 되는 신고 횟수 k가 매개변수로 주어질 때, 각 유저별로 처리 결과 메일을 받은 횟수를 배열에 담아 return 하도록 solution 함수를 완성해주세요.

## 풀이 

해당 문제는 맵을 만들어 이용중인 유저를 키로 주고, 값에 Set객체를 주어  

> Key - A : Value - [ "A를 신고한 유저1", "A를 신고한 유저2" ]  

의 형태로 만들어서 문제에 주어진 k의 값에 따라 정지유무를 판단해 주면 된다.  
Value부분에 Set이 들어가는 이유는 같은 유저에 대한 중복 신고를 방지하기 위해 중복을 허용하지 않는 Collection객체인 Set을 사용했다.

또한, 유저 ID 목록과 출력 답안에 대한 메일 수를 일치시키기 위해 LinkedHashMap을 이용하여 출력 했다.

## 풀이 소스

<script src="https://gist.github.com/dh37789/9b563b680fcf73c8b79c7c54643b9a06.js"></script>
