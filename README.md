제공해주신 프로젝트 보고서(Word 파일)와 파이썬 코드들을 바탕으로, 깃허브(GitHub) README.md에 바로 사용할 수 있도록 정리해 드립니다.

보고서의 논리적 흐름(서론 -> 환경 설정 -> 데이터 수집 -> 전처리 및 분석 -> 시각화 -> 결론)에 맞춰 코드를 분할 배치하고, 각 단계에 대한 설명을 보고서 내용에 기반하여 작성했습니다.
💡 깃허브 업로드 가이드

    이미지 폴더 생성: 깃허브 리포지토리에 images라는 폴더를 만들고, 보고서에 있는 사진 파일들(캡쳐본)을 업로드하세요.

    이미지 경로 연결: 아래 마크다운 내용 중 ![설명](images/사진파일명.png) 부분의 파일명을 실제 업로드한 파일명으로 수정하면 이미지가 정상적으로 나옵니다.

    파일 생성: 리포지토리 루트 경로에 README.md 파일을 만들고 아래 내용을 복사/붙여넣기 하세요.

📊 채용공고 및 기업 리뷰 기반 IT 직군 역량 분석

인하공업전문대학 컴퓨터정보공학과 빅데이터 프로젝트

    작성자: 3학년 B반 202144051 김선재 

    프로젝트 기간: 2024.09 ~ 2024.12

1. 📝 프로젝트 개요 (Introduction)
1.1 주제 선정 이유

대학 졸업 및 취업 시기가 다가옴에 따라, 현재 IT 시장에서 기업들이 실제로 요구하는 기술 트렌드와 인재상을 파악하고자 했습니다. 막연한 스펙 쌓기가 아닌, 데이터에 기반하여 강조해야 할 역량을 분석하기 위해 본 프로젝트를 기획했습니다.

1.2 분석 대상 및 범위

    대상 사이트: 원티드 (Wanted) - 텍스트 위주의 구성으로 크롤링에 용이함 

분석 직군: 개발 직군 전체 (신입 및 경력 1년 미만)

지역: 서울, 인천, 경기도

프로세스: 데이터 수집(크롤링) -> 정제 -> 분석(NLP) -> 시각화

2. ⚙️ 개발 환경 설정 (Environment Setup)

Google Colab 환경에서 진행하였으며, 크롤링 및 텍스트 분석을 위해 Selenium, GLiNER, KoNLPy 등의 라이브러리를 사용했습니다.

Colab의 리눅스 환경에서 일반 Chrome 브라우저는 보안 문제로 실행되지 않으므로, Google에서 배포하는 Stable 버전의 Chrome을 직접 설치하여 사용했습니다.

Bash

# 일반 크롬 브라우저는 보안 문제로 오류. 구글에서 직접 내려 받음
%%shell
sudo apt-get update
sudo apt-get install -y wget unzip

wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt-get -f install -y  # 의존성 오류 발생시 자동 수정

# 한글 폰트 및 필요 라이브러리 설치
!apt-get -qq -y install fonts-nanum > /dev/null
!pip install selenium webdriver_manager gliner konlpy

3. 🕷️ 데이터 수집 (Data Crawling)
3.1 봇 탐지 우회 설정

Selenium을 이용한 백그라운드 실행 시 서버 차단을 방지하기 위해 다음과 같은 설정을 적용했습니다.

    User-Agent 변경: 봇이 아닌 일반 윈도우 환경 브라우저처럼 보이도록 설정 

Automation Flags 제거: 자동화 도구 사용 흔적(Flag) 비활성화

WebDriver 속성 조작: navigator.webdriver 값을 undefined로 강제 설정

Python

# 봇 탐지 우회 및 크롬 GUI 없이 백그라운드 실행 설정
options = Options()
options.add_argument("--headless=new")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")
options.add_argument("--window-size=1920,1080")

# User-Agent 변경
user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
options.add_argument(f'user-agent={user_agent}')

# 자동화 제어 메시지 제거
options.add_experimental_option("excludeSwitches", ["enable-automation"])
options.add_experimental_option('useAutomationExtension', False)
options.add_argument('--disable-blink-features=AutomationControlled')

# 드라이버 실행 & 봇 탐지 방지 스크립트 주입
service = Service(ChromeDriverManager().install())
driver = webdriver.Chrome(service=service, options=options)
driver.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {
    "source": """
    Object.defineProperty(navigator, 'webdriver', {
      get: () => undefined
    })
  """
})

3.2 링크 수집 및 상세 내용 크롤링

스크롤을 내려 로드된 공고들의 링크(href)를 수집하고 중복을 제거했습니다. 이후 각 상세 페이지에 접속하여 BeautifulSoup를 이용해 제목, 회사명, 주요업무, 자격요건, 우대사항을 파싱했습니다.

Python

# 상세 내용 크롤링 로직 (헤더 텍스트 기준 본문 추출)
def get_text_by_header(target_text):
    # h3, h2, strong 태그 중 target_text를 포함하는 것 찾기
    headers = soup.find_all(['h3', 'h2', 'strong'], string=lambda x: x and target_text in x)
    result_text = ""
    for h in headers:
        # 헤더 바로 다음 요소들(p, ul 등)의 텍스트 가져오기
        next_elem = h.find_next_sibling()
        if next_elem:
            result_text += next_elem.get_text(separator=" ", strip=True)
    return result_text

main_work = get_text_by_header("주요업무")
qualification = get_text_by_header("자격요건")
preference = get_text_by_header("우대사항")

4. 🧹 데이터 가공 및 분석 (Data Processing)

수집된 텍스트 데이터를 **기술 스택(Tech Stack)**과 **역량(Soft/Hard Skills)**으로 분류하여 분석했습니다.

4.1 기술 스택 추출 (GLiNER Model)

기술 스택은 트렌드 변화가 빠르므로, 사전 학습된 개체명 인식(NER) 모델인 GLiNER를 사용하여 문맥 내에서 기술 용어를 추출했습니다.

    Labels: Programming Language, Framework & Library, Database & Tool, Cloud & DevOps, AI & Data Stack 

Python

# GLiNER 모델을 이용한 기술 스택 추출
def extract_tech_stack(text, model):
    if not isinstance(text, str) or len(text) < 5: return ""
    
    tech_labels = ["Programming Language", "Framework & Library", "Database & Tool", "Cloud & DevOps", "AI & Data Stack"]
    entities = model.predict_entities(text, tech_labels, threshold=0.3)
    
    found_techs = [entity["text"] for entity in entities]
    return ', '.join(sorted(list(set(found_techs))))

4.2 소프트/하드 스킬 분류 (Rule-based)

역량 키워드는 비교적 정형화되어 있으므로, 특정 키워드 리스트와 문장 끝맺음 패턴(~분, ~능력, ~경험 등)을 활용한 규칙 기반 알고리즘으로 추출했습니다.

Python

# 키워드 기반 역량 문장 추출
def extract_soft_hard_skill(text):
    # ... (중략) ...
    # 키워드 정의
    tech_keyword = ['설계', '구축', '운영', '개발', 'AWS', 'Cloud', 'API', 'DB', 'CI/CD' ...]
    soft_keyword = ['커뮤니케이션', '소통', '협업', '성장', '열정', '주도', '책임' ...]

    for sent in sentences:
        # 문장 끝맺음 패턴 검사
        if not re.search(r'(분|함|력|험|해|식|가능|필수|우대|중시)$', sent): continue
        
        # 키워드 포함 여부에 따라 분류
        is_hard = any(t in sent for t in tech_keyword)
        is_soft = any(t in sent for t in soft_keyword)
        
        if is_soft: soft_skills.append(sent)
        elif is_hard: hard_skills.append(sent)
            
    return " | ".join(soft_skills), " | ".join(hard_skills)

5. 📈 시각화 (Visualization)

추출된 데이터의 빈도수(Frequency)를 Counter로 계산하고, 상위 20개 키워드를 **막대 그래프(Bar Plot)**와 **워드 클라우드(Word Cloud)**로 시각화했습니다. 정확한 분석을 위해 불용어(Stopwords) 처리를 진행했습니다.

5.1 기술 스택 분석 결과

    Python이 압도적으로 많이 요구됨.

    AI 트렌드 반영: PyTorch, TensorFlow가 상위권에 위치. 

Web: Frontend는 React/Next.js, Backend는 Java/Spring, Node.js 강세.

Infra: AWS, Docker, Git 사용 역량 필수.

5.2 태도 및 자세(Soft Skill) 분석 결과

    핵심 키워드: 협업, 커뮤니케이션, 소통 (팀 단위 개발 환경 반영). 

인재상: 주도적, 적극적, 논리적 사고, 비즈니스 이해도.

Python

# 시각화 함수 (Bar Plot & Word Cloud)
def visualize_top_skills(keywords, title, text, color_palette='viridis'):
    # 빈도 계산
    count_data = Counter(keywords)
    df_freq = pd.DataFrame(count_data.most_common(20), columns=['Keyword', 'Frequency'])
    
    # 막대 그래프
    sns.barplot(x='Frequency', y='Keyword', data=df_freq, palette=color_palette)
    # ... (그래프 설정 코드) ...
    plt.show()
    
    # 워드 클라우드
    wc = WordCloud(font_path='...', colormap='coolwarm' if 'Hard' in title else 'summer')
    wc.generate_from_frequencies(count_data)
    plt.imshow(wc, interpolation='bilinear')
    plt.show()

6. 🏁 결론 (Conclusion)

본 프로젝트를 통해 IT 개발 직군 취업 시장에서 다음과 같은 인사이트를 도출했습니다.

    기술 역량: Python 활용 능력과 AI/Data 관련 경험이 중요하며, 웹 분야에서는 React와 **Java(Spring)**가 여전히 강력한 표준입니다. 또한 AWS, Docker, Git은 직군을 불문하고 필수적인 기본 소양입니다. 

협업 역량: 단순 코딩 능력뿐만 아니라 협업과 원활한 커뮤니케이션 능력을 매우 중요하게 평가합니다.

문제 해결 태도: 수동적인 업무 수행보다는 주도적이고 적극적으로 문제를 해결하고, 이를 논리적으로 비즈니스와 연결할 수 있는 역량이 합격의 핵심 요소로 유추됩니다.
