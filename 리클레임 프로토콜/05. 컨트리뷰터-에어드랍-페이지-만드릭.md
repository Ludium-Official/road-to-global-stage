# GitHub 레포지토리 컨트리뷰터 검증 앱 상세 구현 가이드

이 가이드는 Reclaim Protocol을 사용하여 GitHub 레포지토리의 컨트리뷰터를 검증하는 Next.js 애플리케이션을 만드는 전체 과정을 설명합니다.

## 1. 프로젝트 설정

1. 새로운 Next.js 프로젝트를 생성합니다:

   ```bash
   npx create-next-app github-contributor-verification
   cd github-contributor-verification
   ```

2. 필요한 패키지를 설치합니다:

   ```bash
   npm install @reclaimprotocol/js-sdk react-qr-code
   ```

3. `.env.local` 파일을 생성하고 다음 환경 변수를 설정합니다:

   ```
   NEXT_PUBLIC_APP_ID=your_app_id
   APP_SECRET=your_app_secret
   NEXT_PUBLIC_PROVIDER_ID=your_provider_id
   NEXT_PUBLIC_REPO_OWNER=repository_owner
   NEXT_PUBLIC_REPO_NAME=repository_name
   ```

   **주의**: `APP_SECRET`은 클라이언트 측에 노출되지 않도록 주의해야 합니다. 실제 구현에서는 서버 측에서 처리해야 합니다.

## 2. 메인 페이지 구현 (app/page.js)

`app/page.js` 파일을 다음과 같이 작성합니다:

```jsx
"use client";
import React, { useState } from "react";
import { useRouter } from "next/navigation";
import { Reclaim } from "@reclaimprotocol/js-sdk";
import QRCode from "react-qr-code";

const APP_ID = process.env.NEXT_PUBLIC_APP_ID;
const APP_SECRET = process.env.APP_SECRET; // 주의: 실제 구현에서는 서버 측에서 처리해야 합니다.
const PROVIDER_ID = process.env.NEXT_PUBLIC_PROVIDER_ID;
const REPO_OWNER = process.env.NEXT_PUBLIC_REPO_OWNER;
const REPO_NAME = process.env.NEXT_PUBLIC_REPO_NAME;

const GitHubVerification = () => {
  const [url, setUrl] = useState("");
  const [isVerifying, setIsVerifying] = useState(false);
  const [status, setStatus] = useState("");
  const [error, setError] = useState("");
  const [showContributeOption, setShowContributeOption] = useState(false);
  const router = useRouter();

  const reclaimClient = new Reclaim.ProofRequest(APP_ID);

  const handleVerificationSuccess = async (proof) => {
    console.log("Reclaim verification success", proof);
    try {
      const parameters = JSON.parse(proof[0].claimData.parameters);
      const username = parameters.paramValues.username;

      setStatus(
        `GitHub username verified: ${username}. Checking repository contribution...`
      );

      const response = await fetch(
        `/api/github/${REPO_OWNER}/${REPO_NAME}/${username}`
      );

      if (!response.ok) {
        throw new Error("Failed to verify contributor status");
      }

      const result = await response.json();
      if (result.isContributor) {
        setStatus("Verification successful! You are a contributor.");
        router.push("/congrats");
      } else {
        setError("You are not a contributor to the specified repository.");
        setShowContributeOption(true);
      }
    } catch (err) {
      console.error("Server verification failed", err);
      setError("Failed to verify contributor status. Please try again.");
    } finally {
      setIsVerifying(false);
    }
  };

  const handleVerificationFailure = (error) => {
    console.error("Reclaim verification failed", error);
    setError("Verification failed. Please try again.");
    setIsVerifying(false);
  };

  const setupReclaimClient = async () => {
    await reclaimClient.buildProofRequest(PROVIDER_ID);
    reclaimClient.setSignature(
      await reclaimClient.generateSignature(APP_SECRET)
    );
  };

  const startReclaimSession = () => {
    reclaimClient.startSession({
      onSuccessCallback: handleVerificationSuccess,
      onFailureCallback: handleVerificationFailure,
    });
  };

  const generateVerificationRequest = async () => {
    setIsVerifying(true);
    setStatus("Generating verification request...");
    setError("");
    setShowContributeOption(false);
    try {
      reclaimClient.addContext(
        `user's GitHub username`,
        "for repository contribution verification"
      );
      await setupReclaimClient();
      const { requestUrl } = await reclaimClient.createVerificationRequest();
      setUrl(requestUrl);
      startReclaimSession();
      setStatus("Scan the QR code to verify your GitHub username");
    } catch (err) {
      setError("Failed to generate verification request. Please try again.");
      setIsVerifying(false);
    }
  };

  const handleContribute = () => {
    window.open(`https://github.com/${REPO_OWNER}/${REPO_NAME}`, "_blank");
  };

  return (
    <div className="flex flex-col items-center justify-center min-h-screen p-24">
      <h1 className="mb-8 text-4xl font-bold text-center">
        GitHub Repo Contributor Verification
      </h1>
      <p className="mb-8 text-xl text-center">
        Verify your contributions to the {REPO_OWNER}/{REPO_NAME} repository.
      </p>
      <button
        onClick={generateVerificationRequest}
        disabled={isVerifying}
        className="px-4 py-2 font-bold text-white bg-blue-500 rounded hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed mb-4"
      >
        {isVerifying ? "Verifying..." : "Verify GitHub Contributions"}
      </button>
      {status && (
        <div className="mb-4 p-4 bg-blue-100 text-blue-700 border border-blue-200 rounded max-w-md w-full">
          <p className="font-bold">Status</p>
          <p>{status}</p>
        </div>
      )}
      {error && (
        <div className="mb-4 p-4 bg-red-100 text-red-700 border border-red-200 rounded max-w-md w-full">
          <p className="font-bold">Error</p>
          <p>{error}</p>
        </div>
      )}
      {showContributeOption && (
        <div className="mt-4 text-center">
          <p className="mb-2">Want to become a contributor?</p>
          <button
            onClick={handleContribute}
            className="px-4 py-2 font-bold text-white bg-green-500 rounded hover:bg-green-700"
          >
            Contribute Now
          </button>
        </div>
      )}
      {url && (
        <div className="mt-4">
          <QRCode value={url} />
        </div>
      )}
    </div>
  );
};

export default GitHubVerification;
```

## 3. 축하 페이지 구현 (app/congrats/page.js)

`app/congrats/page.js` 파일을 다음과 같이 작성합니다:

```jsx
"use client";
import React from "react";
import { useRouter } from "next/navigation";

const CongratulationsPage = () => {
  const router = useRouter();

  const handleStartClick = () => {
    alert("환영합니다! 이제 앱의 모든 기능을 사용하실 수 있습니다.");
    // 여기에 메인 앱 페이지로 리다이렉트하는 로직을 추가할 수 있습니다.
  };

  return (
    <div className="flex flex-col items-center justify-center min-h-screen p-4 bg-gradient-to-r from-green-400 to-blue-500">
      <div className="text-center">
        <h1 className="mb-6 text-4xl font-bold text-white md:text-6xl">
          축하합니다!
        </h1>
        <p className="mb-8 text-xl text-white md:text-2xl">
          GitHub 레포지토리 컨트리뷰터 검증에 성공하셨습니다.
        </p>
        <p className="mb-8 text-lg text-white md:text-xl">
          귀하의 기여에 감사드립니다. 이제 특별한 기능에 접근하실 수 있습니다.
        </p>
        <button
          className="px-6 py-3 mt-8 font-semibold text-blue-600 transition duration-300 bg-white rounded-full shadow-lg hover:bg-blue-100 hover:scale-105 active:scale-95"
          onClick={handleStartClick}
        >
          시작하기
        </button>
      </div>
    </div>
  );
};

export default CongratulationsPage;
```

## 4. GitHub API 연동 (app/api/github/[[...params]]/route.js)

`app/api/github/[[...params]]/route.js` 파일을 다음과 같이 작성합니다:

```javascript
import { NextResponse } from "next/server";

const GITHUB_API_URL = "https://api.github.com";

export async function GET(request, { params }) {
  const { params: urlParams } = params;

  if (!urlParams || urlParams.length < 2) {
    return NextResponse.json(
      { error: "Owner and repo parameters are required" },
      { status: 400 }
    );
  }

  const [owner, repo, contributor] = urlParams;

  try {
    const contributors = await fetchAllContributors(owner, repo);

    if (contributor) {
      const isContributor = contributors.some(
        (c) => c.login.toLowerCase() === contributor.toLowerCase()
      );
      return NextResponse.json({ isContributor });
    }

    return NextResponse.json(contributors);
  } catch (error) {
    console.error("Error fetching contributors:", error);
    return NextResponse.json(
      { error: "Failed to fetch contributors" },
      { status: 500 }
    );
  }
}

async function fetchAllContributors(
  owner,
  repo,
  page = 1,
  perPage = 100,
  allContributors = []
) {
  const response = await fetch(
    `${GITHUB_API_URL}/repos/${owner}/${repo}/contributors?page=${page}&per_page=${perPage}`,
    {
      headers: {
        Accept: "application/vnd.github.v3+json",
        "User-Agent": "NextJS-App",
      },
    }
  );

  if (!response.ok) {
    throw new Error(`GitHub API responded with status ${response.status}`);
  }

  const contributors = await response.json();
  allContributors.push(...contributors);

  const linkHeader = response.headers.get("Link");
  if (linkHeader && linkHeader.includes('rel="next"')) {
    return fetchAllContributors(
      owner,
      repo,
      page + 1,
      perPage,
      allContributors
    );
  }

  return allContributors;
}
```

## 5. 주요 기능 설명

### Reclaim Protocol 사용

- `reclaimClient`: Reclaim SDK를 초기화하여 GitHub 사용자 이름을 확인합니다.
- `setupReclaimClient`: Reclaim 클라이언트를 설정하고 서명을 생성합니다.
- `startReclaimSession`: Reclaim 세션을 시작하고 콜백 함수를 설정합니다.
- `generateVerificationRequest`: 검증 요청을 생성하고 QR 코드 URL을 설정합니다.

### GitHub API 연동

- `fetchAllContributors`: GitHub API를 사용하여 레포지토리의 모든 컨트리뷰터를 가져옵니다.
- 페이지네이션을 처리하여 대규모 레포지토리의 모든 컨트리뷰터를 확인합니다.

### 사용자 경험

- 상태 관리: 검증 과정의 각 단계를 사용자에게 표시합니다.
- 에러 처리: 오류 발생 시 사용자에게 명확한 메시지를 제공합니다.
- QR 코드: 모바일 기기에서 쉽게 스캔할 수 있도록 QR 코드를 생성합니다.

## 6. 구현 시 주의사항

1. 보안:

   - `APP_SECRET`은 반드시 서버 측에서 관리해야 합니다. 이 예제에서는 학습을 위해 클라이언트 측 코드에 포함되어 있지만, 실제 구현에서는 절대 이렇게 하면 안 됩니다.
   - API 키와 같은 민감한 정보는 환경 변수로 관리하세요.

2. 성능:
   - GitHub API 호출 결과를 캐싱하여 반복적인 요청을 줄이는 것을 고려하세요.
   - 대규모 레포지토리의 경우 컨트리뷰터 목록 가져오기에 시간이 걸릴 수 있으므로, 백그라운드 작업으로 처리하는 방법을 고려하세요.
