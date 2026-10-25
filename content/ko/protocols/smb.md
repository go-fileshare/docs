---
title: "SMB — NTLMv2"
linkTitle: "SMB: NTLMv2"
weight: 10
description: "NTLMv2를 쓰는 SMB: 디렉터리가 보유해야 하는 것, 격리 상태에서의 권한이 필요한 포트, 디렉터리 다시 불러오기가 열린 연결에 미치는 영향."
tags: [프로토콜, smb]
---

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

비밀번호는 **네트워크를 건너지 않습니다**. 클라이언트는 비밀번호로 계산한 증명을
보냅니다. 공유는 읽는 사람에게 나중에 쓰기가 실패하는 방식이 아니라, 시도하기
전에 접근 마스크로 읽기 전용임을 알려 줍니다.

## 디렉터리가 보유해야 하는 것 {#what-a-directory-must-hold}

NTLMv2에는 비밀번호 자체 또는 그 MD4(`sambaNTPassword`, `nt_hash` 열)가
필요합니다. **다른 것으로는 안 됩니다.**

비밀번호를 *확인*하기만 하는 디렉터리(LDAP 바인드, bcrypt 열)는 확인이 아무리
훌륭해도 SMB에 응답할 수 없습니다. 서버가 클라이언트가 계산한 것과 같은 증명을
계산해야 하기 때문입니다. 이는 이 프로그램의 한계가 아니라 프로토콜의 속성입니다.

[`check`]({{< relref "/configuration/check.md" >}})는 이런 사용자에 대해 구성 시점에
SMB 열에 `-`를 출력하며, 마운트할 때 비로소 알게 되도록 두지 않습니다.

## 권한 있는 서버 없이 권한이 필요한 포트 {#a-privileged-port-without-a-privileged-server}

Unix에서 [`--isolate`]({{< relref "/operations/isolation.md" >}}) 아래에서는 부모가
445를 바인딩하고 리스너를 자식에게 넘기므로, 실제로 SMB를 처리하는 프로세스는
권한이 전혀 필요 없습니다. Windows에는 `ExtraFiles`가 없으므로 자식이 직접 주소를
바인딩합니다. 격리는 같지만, 권한이 필요한 포트에 관한 절반은 해당하지 않습니다.

## 이 프로젝트가 작성하지 않은 클라이언트로 검증 {#verified-against-a-client-this-project-did-not-write}

[go-smb2](https://github.com/cloudsoda/go-smb2)가 CI에서 `-race`로 SMB를 통한
동시 읽기 스무 건과 WebDAV를 통한 스무 건을 함께 구동합니다. macOS는 공유를
SMB로 마운트하고 `curl`은 같은 이미지에 WebDAV로 쓴 다음, WebDAV가 쓴 내용을
다시 읽어 옵니다.

## 사용자가 바뀔 때 {#when-the-people-change}

SMB는 **공유가 추가될 때** 누가 그 공유에 연결할 수 있는지를 확정하고, 트리
연결마다 한 번 검사합니다. 따라서 [디렉터리 다시 불러오기]({{< relref "/administration/reload.md" >}})가
찾아낸 새 사용자는 SMB의 실행 중인 서버를 포함해 제자리에서 추가되며 어떤 연결도
건드리지 않습니다. 그러나 사용자, 자격 증명, 공유의 확장된 목록 등 무엇이든
제거되는 것은 새 세대이며 SMB의 열린 연결은 닫히므로, 폐기가 이미 열린 세션에도
미칩니다. `allow` 그룹이 비어 버린 공유는 **SMB로 아예 제공되지 않습니다**. 빈
`AllowUsers`는 모든 사람으로 해석되기 때문입니다.
