"""
AlitaMed 지원사업 공고 자동 수집기
매주 월요일 GitHub Actions에서 실행
"""

import os
import re
import smtplib
import logging
from datetime import datetime, timedelta
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

import requests
from bs4 import BeautifulSoup

# ─────────────────────────────────────────────
# 로깅 설정
# ─────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
log = logging.getLogger(__name__)

# ─────────────────────────────────────────────
# 공통 설정
# ─────────────────────────────────────────────
HEADERS = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/124.0.0.0 Safari/537.36"
    )
}
TIMEOUT = 15
KEYWORDS = [
    "의료기기", "의료", "R&D", "연구개발", "스타트업", "창업",
    "중소기업", "벤처", "기술개발", "경기도", "용인", "바이오",
    "IVL", "혈관", "심혈관", "카테터", "의료기술",
]

# 오늘 기준 최근 7일
CUTOFF_DATE = datetime.today() - timedelta(days=7)


def is_relevant(title: str) -> bool:
    """제목에 관련 키워드 포함 여부 확인"""
    return any(kw in title for kw in KEYWORDS)


def safe_get(url: str, **kwargs) -> requests.Response | None:
    """requests.get 래퍼 — 실패 시 None 반환"""
    try:
        resp = requests.get(url, headers=HEADERS, timeout=TIMEOUT, **kwargs)
        resp.raise_for_status()
        return resp
    except Exception as e:
        log.warning(f"[GET 실패] {url} → {e}")
        return None


# ─────────────────────────────────────────────
# 사이트별 크롤러
# ─────────────────────────────────────────────

def scrape_bizinfo() -> list[dict]:
    """기업마당 (중소벤처기업부)"""
    results = []
    base = "https://www.bizinfo.go.kr"
    url = f"{base}/sii/siia/selectSIIA200View.do"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    rows = soup.select("table tbody tr")
    for row in rows:
        cells = row.find_all("td")
        if len(cells) < 3:
            continue
        a_tag = row.find("a")
        title = a_tag.get_text(strip=True) if a_tag else ""
        if not title or not is_relevant(title):
            continue
        href = a_tag.get("href", "") if a_tag else ""
        link = href if href.startswith("http") else base + href
        results.append({"site": "기업마당", "title": title, "link": link})
    log.info(f"[기업마당] {len(results)}건")
    return results


def scrape_ntis() -> list[dict]:
    """NTIS 국가 R&D 통합공고"""
    results = []
    base = "https://www.ntis.go.kr"
    url = f"{base}/rndgate/eg/un/ra/mng.do"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    rows = soup.select("table tbody tr, ul.list li, .board-list li")
    for row in rows:
        a_tag = row.find("a")
        if not a_tag:
            continue
        title = a_tag.get_text(strip=True)
        if not title or not is_relevant(title):
            continue
        href = a_tag.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "NTIS(국가 R&D 통합공고)", "title": title, "link": link})
    log.info(f"[NTIS] {len(results)}건")
    return results


def scrape_iris() -> list[dict]:
    """IRIS 범부처통합연구지원시스템"""
    results = []
    base = "https://www.iris.go.kr"
    url = f"{base}/main.do"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select("a"):
        title = a.get_text(strip=True)
        if len(title) < 10 or not is_relevant(title):
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "IRIS(범부처통합연구지원시스템)", "title": title, "link": link})
    # 중복 제거
    seen = set()
    uniq = []
    for r in results:
        if r["title"] not in seen:
            seen.add(r["title"])
            uniq.append(r)
    log.info(f"[IRIS] {len(uniq)}건")
    return uniq[:10]


def scrape_kised() -> list[dict]:
    """창업진흥원"""
    results = []
    base = "https://www.kised.or.kr"
    url = f"{base}/index.es?sid=a1"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select(".notice-list a, .board-list a, ul.list a"):
        title = a.get_text(strip=True)
        if len(title) < 5 or not is_relevant(title):
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "창업진흥원", "title": title, "link": link})
    log.info(f"[창업진흥원] {len(results)}건")
    return results


def scrape_egbiz() -> list[dict]:
    """경기기업비서 (이지비즈)"""
    results = []
    base = "https://www.egbiz.or.kr"
    url = f"{base}/index.do"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select("a"):
        title = a.get_text(strip=True)
        if len(title) < 8 or not is_relevant(title):
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "경기기업비서(이지비즈)", "title": title, "link": link})
    seen = set()
    uniq = []
    for r in results:
        if r["title"] not in seen:
            seen.add(r["title"])
            uniq.append(r)
    log.info(f"[경기기업비서] {len(uniq)}건")
    return uniq[:10]


def scrape_gbsa() -> list[dict]:
    """경기도경제과학진흥원"""
    results = []
    base = "https://www.gbsa.or.kr"
    url = base + "/"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select("a"):
        title = a.get_text(strip=True)
        if len(title) < 8 or not is_relevant(title):
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "경기도경제과학진흥원", "title": title, "link": link})
    seen = set()
    uniq = []
    for r in results:
        if r["title"] not in seen:
            seen.add(r["title"])
            uniq.append(r)
    log.info(f"[경기도경제과학진흥원] {len(uniq)}건")
    return uniq[:10]


def scrape_tipa() -> list[dict]:
    """중소기업기술정보진흥원 (TIPA)"""
    results = []
    base = "https://www.tipa.or.kr"
    url = base + "/"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select("a"):
        title = a.get_text(strip=True)
        if len(title) < 8 or not is_relevant(title):
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "TIPA(중소기업기술정보진흥원)", "title": title, "link": link})
    seen = set()
    uniq = []
    for r in results:
        if r["title"] not in seen:
            seen.add(r["title"])
            uniq.append(r)
    log.info(f"[TIPA] {len(uniq)}건")
    return uniq[:10]


def scrape_kstartup() -> list[dict]:
    """K-스타트업 창업지원포털"""
    results = []
    base = "https://www.k-startup.go.kr"
    url = base + "/"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select("a"):
        title = a.get_text(strip=True)
        if len(title) < 8 or not is_relevant(title):
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "K-스타트업 창업지원포털", "title": title, "link": link})
    seen = set()
    uniq = []
    for r in results:
        if r["title"] not in seen:
            seen.add(r["title"])
            uniq.append(r)
    log.info(f"[K-스타트업] {len(uniq)}건")
    return uniq[:10]


def scrape_yongin() -> list[dict]:
    """용인시 기업지원시스템"""
    results = []
    base = "https://ybs.ypa.or.kr"
    url = f"{base}/portal.do"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select("a"):
        title = a.get_text(strip=True)
        if len(title) < 8 or not is_relevant(title):
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "용인시 기업지원시스템", "title": title, "link": link})
    seen = set()
    uniq = []
    for r in results:
        if r["title"] not in seen:
            seen.add(r["title"])
            uniq.append(r)
    log.info(f"[용인시] {len(uniq)}건")
    return uniq[:10]


def scrape_emedi() -> list[dict]:
    """의료기기안심책방 (임상시험 검색)"""
    results = []
    base = "https://emedi.mfds.go.kr"
    url = base + "/"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select("a"):
        title = a.get_text(strip=True)
        if len(title) < 8:
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "의료기기안심책방(의료기기 통합정보)", "title": title, "link": link})
    seen = set()
    uniq = []
    for r in results:
        if r["title"] not in seen:
            seen.add(r["title"])
            uniq.append(r)
    log.info(f"[의료기기안심책방] {len(uniq)}건")
    return uniq[:10]


def scrape_unicornbridge() -> list[dict]:
    """경기창조경제혁신센터"""
    results = []
    base = "https://unicornbridge.kr"
    url = base + "/"
    resp = safe_get(url)
    if not resp:
        return results
    soup = BeautifulSoup(resp.text, "html.parser")
    for a in soup.select("a"):
        title = a.get_text(strip=True)
        if len(title) < 8 or not is_relevant(title):
            continue
        href = a.get("href", "")
        link = href if href.startswith("http") else base + href
        results.append({"site": "경기창조경제혁신센터", "title": title, "link": link})
    seen = set()
    uniq = []
    for r in results:
        if r["title"] not in seen:
            seen.add(r["title"])
            uniq.append(r)
    log.info(f"[경기창조경제혁신센터] {len(uniq)}건")
    return uniq[:10]


# ─────────────────────────────────────────────
# 전체 수집
# ─────────────────────────────────────────────

def collect_all() -> list[dict]:
    scrapers = [
        scrape_bizinfo,
        scrape_ntis,
        scrape_iris,
        scrape_kised,
        scrape_egbiz,
        scrape_gbsa,
        scrape_tipa,
        scrape_kstartup,
        scrape_yongin,
        scrape_emedi,
        scrape_unicornbridge,
    ]
    all_results = []
    for fn in scrapers:
        try:
            all_results.extend(fn())
        except Exception as e:
            log.error(f"[{fn.__name__}] 크롤링 오류: {e}")
    return all_results


# ─────────────────────────────────────────────
# 이메일 빌드
# ─────────────────────────────────────────────

SITE_URLS = {
    "기업마당":                   "https://www.bizinfo.go.kr/sii/siia/selectSIIA200View.do",
    "NTIS(국가 R&D 통합공고)":    "https://www.ntis.go.kr/rndgate/eg/un/ra/mng.do",
    "IRIS(범부처통합연구지원시스템)": "https://www.iris.go.kr/main.do",
    "창업진흥원":                  "https://www.kised.or.kr/index.es?sid=a1",
    "경기기업비서(이지비즈)":       "https://www.egbiz.or.kr/index.do",
    "경기도경제과학진흥원":         "https://www.gbsa.or.kr/",
    "TIPA(중소기업기술정보진흥원)": "https://www.tipa.or.kr/",
    "K-스타트업 창업지원포털":      "https://www.k-startup.go.kr/",
    "용인시 기업지원시스템":        "https://ybs.ypa.or.kr/portal.do",
    "의료기기안심책방(의료기기 통합정보)": "https://emedi.mfds.go.kr/",
    "경기창조경제혁신센터":         "https://unicornbridge.kr/",
}


def build_email_html(items: list[dict]) -> str:
    today_str = datetime.today().strftime("%Y년 %m월 %d일")

    # 사이트별 그룹화
    grouped: dict[str, list[dict]] = {}
    for item in items:
        grouped.setdefault(item["site"], []).append(item)

    # 공고 블록 생성
    if grouped:
        blocks = ""
        for site, posts in grouped.items():
            site_url = SITE_URLS.get(site, "#")
            post_rows = ""
            for p in posts:
                post_rows += f"""
                <tr>
                  <td style="padding:8px 12px; border-bottom:1px solid #f0f0f0; font-size:14px; color:#333;">
                    · <a href="{p['link']}" style="color:#1a5fa8; text-decoration:none;">{p['title']}</a>
                  </td>
                </tr>"""
            blocks += f"""
            <div style="margin-bottom:24px;">
              <div style="background:#1a5fa8; color:#fff; padding:10px 16px;
                          border-radius:6px 6px 0 0; font-size:15px; font-weight:bold;">
                🏢 {site}
              </div>
              <table width="100%" cellpadding="0" cellspacing="0"
                     style="border:1px solid #dce3ec; border-top:none; border-radius:0 0 6px 6px; background:#fff;">
                {post_rows}
                <tr>
                  <td style="padding:8px 12px; font-size:13px; color:#888;">
                    🔗 사이트 바로가기: <a href="{site_url}" style="color:#1a5fa8;">{site_url}</a>
                  </td>
                </tr>
              </table>
            </div>"""
    else:
        blocks = """
        <div style="text-align:center; padding:40px; color:#999; font-size:15px;">
          이번 주에는 수집된 관련 공고가 없습니다.<br>
          키워드 또는 대상 사이트를 확인해 주세요.
        </div>"""

    html = f"""<!DOCTYPE html>
<html lang="ko">
<head><meta charset="UTF-8"></head>
<body style="margin:0; padding:0; background:#f4f6f9; font-family:'Apple SD Gothic Neo',
             'Malgun Gothic', sans-serif;">

  <table width="100%" cellpadding="0" cellspacing="0" style="background:#f4f6f9; padding:30px 0;">
    <tr>
      <td align="center">
        <table width="660" cellpadding="0" cellspacing="0"
               style="background:#fff; border-radius:10px;
                      box-shadow:0 2px 12px rgba(0,0,0,.08); overflow:hidden;">

          <!-- 헤더 -->
          <tr>
            <td style="background:#1a5fa8; padding:28px 32px;">
              <p style="margin:0; color:#fff; font-size:22px; font-weight:bold;">
                📢 AlitaMed 지원사업 공고 알리미
              </p>
              <p style="margin:6px 0 0; color:#cce0ff; font-size:13px;">
                수집 기준일 : {today_str}
              </p>
            </td>
          </tr>

          <!-- 인사말 -->
          <tr>
            <td style="padding:28px 32px 16px;">
              <p style="margin:0; font-size:15px; line-height:1.8; color:#333;">
                안녕하세요, 알리타메드 임직원 여러분.<br>
                신규 지원사업 공고를 요약해주는 알리미 봇입니다.<br>
                업무에 참고하시기 바랍니다!
              </p>
              <hr style="border:none; border-top:2px solid #e8edf5; margin:20px 0;">
            </td>
          </tr>

          <!-- 공고 목록 -->
          <tr>
            <td style="padding:0 32px 24px;">
              <p style="margin:0 0 16px; font-size:16px; font-weight:bold; color:#1a5fa8;">
                📋 이번 주 수집 공고 (총 {len(items)}건)
              </p>
              {blocks}
            </td>
          </tr>

          <!-- 맺음말 -->
          <tr>
            <td style="padding:20px 32px 32px;">
              <div style="background:#f0f5ff; border-left:4px solid #1a5fa8;
                          padding:16px 20px; border-radius:4px;">
                <p style="margin:0; font-size:14px; line-height:1.9; color:#333;">
                  팀별로 검토가 필요한 내용이 있는지 살펴봐 주세요!<br>
                  해당 공고와 관련해 문의사항 있으시면, 언제든 봇에게 말씀해 주세요!<br>
                  <strong>감사합니다!</strong>
                </p>
              </div>
            </td>
          </tr>

          <!-- 푸터 -->
          <tr>
            <td style="background:#f4f6f9; padding:16px 32px; text-align:center;">
              <p style="margin:0; font-size:12px; color:#aaa;">
                AlitaMed 지원사업 자동 알리미 봇 · GitHub Actions 자동 발송
              </p>
            </td>
          </tr>

        </table>
      </td>
    </tr>
  </table>

</body>
</html>"""
    return html


# ─────────────────────────────────────────────
# 이메일 발송
# ─────────────────────────────────────────────

def send_email(html_body: str, item_count: int) -> None:
    smtp_host = os.environ["SMTP_HOST"]          # e.g. smtp.gmail.com
    smtp_port = int(os.environ.get("SMTP_PORT", 587))
    smtp_user = os.environ["SMTP_USER"]          # 발신 계정
    smtp_pass = os.environ["SMTP_PASS"]          # 앱 비밀번호
    recipients = os.environ["MAIL_TO"].split(",") # 수신자 (콤마 구분)

    today_str = datetime.today().strftime("%Y.%m.%d")
    subject = f"[AlitaMed 알리미] {today_str} 신규 지원사업 공고 ({item_count}건)"

    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"] = smtp_user
    msg["To"] = ", ".join(recipients)
    msg.attach(MIMEText(html_body, "html", "utf-8"))

    with smtplib.SMTP(smtp_host, smtp_port) as server:
        server.starttls()
        server.login(smtp_user, smtp_pass)
        server.sendmail(smtp_user, recipients, msg.as_string())

    log.info(f"이메일 발송 완료 → {recipients} ({item_count}건)")


# ─────────────────────────────────────────────
# 엔트리포인트
# ─────────────────────────────────────────────

if __name__ == "__main__":
    log.info("=== AlitaMed 지원사업 수집 시작 ===")
    items = collect_all()
    log.info(f"총 수집 건수: {len(items)}건")

    html = build_email_html(items)
    send_email(html, len(items))

    log.info("=== 완료 ===")
