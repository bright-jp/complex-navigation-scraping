# 複雑なナビゲーションを持つWebサイトのスクレイピング

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.jp/) 

このガイドでは、Selenium とブラウザ自動化を使用して、動的ページネーション、無限スクロール、「Load More」ボタンなどの複雑なナビゲーションパターンを持つWebサイトをスクレイピングする方法を解説します。Selenium とブラウザ自動化を使用します。

- [複雑なナビゲーションとは何と見なされますか？](#what-is-considered-complex-navigation)
- [複雑なナビゲーションWebサイトに対応するツール](#tools-to-handle-complex-navigation-websites)
- [一般的な複雑ナビゲーションパターンのスクレイピング](#scraping-common-complex-navigation-patterns)
  - [動的ページネーション](#dynamic-pagination)
  - [‘Load More’ ボタン](#load-more-button)
  - [無限スクロール](#infinite-scrolling)
- [結論](#conclusion)

## 複雑なナビゲーションとは何と見なされますか？

Webスクレイピングにおいて、複雑なナビゲーションとは、コンテンツやページに容易にアクセスできないWebサイト構造を指します。複雑なナビゲーションのシナリオでは、動的要素、非同期のデータ読み込み、またはユーザー主導のインタラクションが関与することが多いです。これらの要素はユーザー体験を向上させる一方で、データ抽出を大幅に複雑にします。よくある例は次のとおりです。

- **JavaScriptレンダリングのナビゲーション**: JavaScriptフレームワークに依存してブラウザ内で直接コンテンツを生成するWebサイトです。
- **ページ分割されたコンテンツ**: データが複数ページに分散しており、ページネーションが AJAX により動的に読み込まれるサイトです。
- **無限スクロール**: ユーザーがスクロールするたびに追加コンテンツを動的に読み込むページです。ソーシャルメディアのフィード、Discourseベースのフォーラム、ニュースサイトなどで典型的です。
- **多階層メニュー**: ネストされたメニューを持ち、より深いナビゲーション階層を表示するために複数回のクリックやホバー操作が必要なサイトです。マーケットプレイスのプロダクトカテゴリツリーなどで一般的です。
- **インタラクティブな地図インターフェース**: 地図やグラフ上にデータを表示し、ユーザーのパンやズームに応じて情報が動的に読み込まれるWebサイトです。
- **タブまたはアコーディオン**: 動的にレンダリングされるタブや折りたたみ式アコーディオンの下にコンテンツが隠れており、サーバーから返されるHTMLに直接埋め込まれていないページです。
- **動的フィルターと並び替えオプション**: 複雑なフィルタリングシステムを持ち、複数のフィルター適用によりURL構造を変更せずにアイテム一覧が動的に再読み込みされるサイトです。

## 複雑なナビゲーションWebサイトに対応するツール

上記の複雑なインタラクションの多くは JavaScript の実行を必要とします。これはブラウザだけが実行できることです。つまり、そのようなページでは単純な [HTML parsers](https://brightdata.jp/blog/web-data/best-html-parsers) に頼ることはできません。代わりに、Selenium、Playwright、Puppeteer のようなブラウザ自動化ツールを使用する必要があります。これらのソリューションにより、ブラウザに対してWebページ上で特定のアクションを実行するようプログラムで指示でき、ユーザー行動を模倣できます。

## 一般的な複雑ナビゲーションパターンのスクレイピング

このガイドでは、複雑なナビゲーションパターンのうち、次の3種類を扱います。

- **動的ページネーション**: AJAX により動的に読み込まれるページ分割データを持つサイトです。
- **‘Load More’ ボタン**: JavaScriptベースのナビゲーションの一般的な例です。
- **無限スクロール**: ユーザーが下にスクロールするにつれて継続的にデータを読み込むページです。

Python の Selenium を使用しますが、ロジックは Playwright、Puppeteer、または他のブラウザ自動化ツールにも適用できます。また、このガイドでは、すでに [web scraping using Selenium](https://brightdata.jp/blog/how-tos/using-selenium-for-web-scraping) の基礎を理解していることを前提としています。

### 動的ページネーション

スクレイピング用サンドボックスとして、「[Oscar Winning Films: AJAX and Javascript](https://www.scrapethissite.com/pages/ajax-javascript/#2014)」を使用します。

![The target page. Note how pagination data is loaded dynamically](https://github.com/luminati-io/complex-navigation-scraping/blob/main/Images/Dynamic-pagniation-example-1536x752.gif)

このサイトは、年ごとにページ分割されたアカデミー賞受賞作品データを動的に読み込みます。

このようなページを効果的にナビゲートしてスクレイピングするには、次の手順に従う必要があります。

1. 新しい年をクリックしてデータ読み込みをトリガーします（ローダー要素が表示されます）。
2. ローダー要素が消えるのを待ちます（データが完全に読み込まれたことを示します）。
3. データを含むテーブルがページ上で正しくレンダリングされたことを確認します。
4. データが利用可能になったらスクレイピングします。

以下は、Python の Selenium を使ってこのロジックを実装する例です。

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options

# Set up Chrome options for headless mode
options = Options()
options.add_argument("--headless")

# Create a Chrome web driver instance
driver = webdriver.Chrome(service=Service(), options=options)

# Connect to the target page
driver.get("https://www.scrapethissite.com/pages/ajax-javascript/")

# Click the "2012" pagination button
element = driver.find_element(By.ID, "2012")
element.click()

# Wait until the loader is no longer visible
WebDriverWait(driver, 10).until(
    lambda d: d.find_element(By.CSS_SELECTOR, "#loading").get_attribute("style") == "display: none;"
)

# Data should now be loaded...

# Wait for the table to be present on the page
WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.CSS_SELECTOR, ".table"))
)

# Where to store the scraped data
films = []

# Scrape data from the table
table_body = driver.find_element(By.CSS_SELECTOR, "#table-body")
rows = table_body.find_elements(By.CSS_SELECTOR, ".film")
for row in rows:
    title = row.find_element(By.CSS_SELECTOR, ".film-title").text
    nominations = row.find_element(By.CSS_SELECTOR, ".film-nominations").text
    awards = row.find_element(By.CSS_SELECTOR, ".film-awards").text
    best_picture_icon = row.find_element(By.CSS_SELECTOR, ".film-best-picture").find_elements(By.TAG_NAME, "i")
    best_picture = True if best_picture_icon else False

    # Store the scraped data
    films.append({
      "title": title,
      "nominations": nominations,
      "awards": awards,
      "best_picture": best_picture
    })

# Data export logic...

# Close the browser driver
driver.quit()
```

このコードスニペットの内訳は次のとおりです。

1.  コードはヘッドレスの Chrome インスタンスをセットアップします。
2.  スクリプトは対象ページを開き、「2012」のページネーションボタンをクリックしてデータ読み込みをトリガーします。
3.  Selenium は [`WebDriverWait()`](https://selenium-python.readthedocs.io/waits.html) を使用してローダーが消えるのを待ちます。
4.  ローダーが消えた後、スクリプトはテーブルが表示されるのを待ちます。
5.  データが完全に読み込まれた後、スクリプトは作品タイトル、ノミネート数、受賞数、作品賞受賞の有無などの詳細を抽出します。抽出した情報は辞書のリストに保存されます。

結果は次のようになります。

```json
[
  {
    "title": "Argo",
    "nominations": "7",
    "awards": "3",
    "best_picture": true
  },
  // ...
  {
    "title": "Curfew",
    "nominations": "1",
    "awards": "1",
    "best_picture": false
  }
]
```

このナビゲーションパターンの扱いに、常に単一の最良アプローチがあるとは限らない点に留意してください。ページの挙動によっては代替手法が必要になる場合があります。例をいくつか挙げます。

*   `WebDriverWait()` を expected conditions と組み合わせて、特定のHTML要素の表示/非表示を待ちます。
*   AJAX のリクエストトラフィックを監視して、新しいコンテンツが取得されたタイミングを検出します。これにはブラウザログの利用が関与する場合があります。
*   ページネーションによってトリガーされるAPIリクエストを特定し、直接リクエストしてプログラム的にデータを取得します（例: [`requests` library](https://brightdata.jp/blog/web-data/python-requests-guide) を使用）。

### ‘Load More’ ボタン

ユーザー操作を伴う JavaScript ベースの複雑なナビゲーションシナリオを示すために、「Load More」ボタンの例を使用します。概念は単純で、アイテムのリストが表示されており、ボタンをクリックすると追加アイテムが読み込まれます。

今回は、Scraping Course の [‘Load More’ example](https://www.scrapingcourse.com/button-click) ページを対象サイトとします。

![The ‘Load More’ target page in action](https://github.com/luminati-io/complex-navigation-scraping/blob/main/Images/Clicking-on-the-load-more-button-1536x752.gif)

この複雑ナビゲーションのスクレイピングパターンに対応するには、次の手順に従ってください。

1.  ‘Load More’ ボタンを見つけてクリックします。
2.  新しい要素がページに読み込まれるのを待ちます。

Selenium で使用するコードは次のとおりです。

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait

# Set up Chrome options for headless mode
options = Options()
options.add_argument("--headless")

# Create a Chrome web driver instance
driver = webdriver.Chrome(options=options)

# Connect to the target page
driver.get("https://www.scrapingcourse.com/button-click")

# Collect the initial number of products
initial_product_count = len(driver.find_elements(By.CSS_SELECTOR, ".product-item"))

# Locate the "Load More" button and click it
load_more_button = driver.find_element(By.CSS_SELECTOR, "#load-more-btn")
load_more_button.click()

# Wait until the number of product items on the page has increased
WebDriverWait(driver, 10).until(lambda driver: len(driver.find_elements(By.CSS_SELECTOR, ".product-item")) > initial_product_count)

# Where to store the scraped data
products = []

# Scrape product details
product_elements = driver.find_elements(By.CSS_SELECTOR, ".product-item")
for product_element in product_elements:
    # Extract product details
    name = product_element.find_element(By.CSS_SELECTOR, ".product-name").text
    image = product_element.find_element(By.CSS_SELECTOR, ".product-image").get_attribute("src")
    price = product_element.find_element(By.CSS_SELECTOR, ".product-price").text
    url = product_element.find_element(By.CSS_SELECTOR, "a").get_attribute("href")

    # Store the scraped data
    products.append({
        "name": name,
        "image": image,
        "price": price,
        "url": url
    })

# Data export logic...

# Close the browser driver
driver.quit()
```

‘Load More’ ボタンのナビゲーションパターンに対応するために、このスクリプトは次を行います。

1.  ページ上の初期プロダクト数を記録します
2.  「Load More」ボタンをクリックします
3.  プロダクト数が増えるまで待機し、新しいアイテムが追加されたことを確認します

このアプローチは、読み込まれる要素数を正確に把握する必要がなくなるため、効率的かつ汎用的です。ただし、代替手法でも同様の結果を得られます。

### 無限スクロール

無限スクロールは、ユーザーエンゲージメントを高めるためにソーシャルメディアやEコマースプラットフォームで広く使われている人気のインタラクションです。この場合、対象は上記と同じページですが、[‘Load More’ ボタンの代わりに無限スクロール](https://www.scrapingcourse.com/infinite-scrolling) を使用します。

![infinite scrolling instead of a 'Load More' button](https://github.com/luminati-io/complex-navigation-scraping/blob/main/Images/Infinite-scrolling-example-1024x501.gif)

多くのブラウザ自動化ツールは、ページを上下にスクロールするための直接的なメソッドを提供しておらず、Selenium も例外ではありません。代わりに、ページ上で JavaScript を実行してスクロール操作を行う必要があります。

解決策は、下方向にスクロールするカスタム JavaScript を書くことです。

1.  指定回数だけ、または
2.  追加データがこれ以上読み込めなくなるまで

> **Note**:\
> 各スクロールで新しいデータが読み込まれ、ページ上の要素数が増加します。

その後、新しく読み込まれたコンテンツをスクレイピングできます。

以下は、Selenium で無限スクロールを行うためのコードです。

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait

# Set up Chrome options for headless mode
options = Options()
# options.add_argument("--headless")

# Create a Chrome web driver instance
driver = webdriver.Chrome(options=options)

# Connect to the target page with infinite scrolling
driver.get("https://www.scrapingcourse.com/infinite-scrolling")

# Current page height
scroll_height = driver.execute_script("return document.body.scrollHeight")
# Number of products on the page
product_count = len(driver.find_elements(By.CSS_SELECTOR, ".product-item"))

# Max number of scrolls
max_scrolls = 10
scroll_count = 1

# Limit the number of scrolls to 10
while scroll_count < max_scrolls:
    # Scroll down
    driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")

    # Wait until the number of product items on the page has increased
    WebDriverWait(driver, 10).until(lambda driver: len(driver.find_elements(By.CSS_SELECTOR, ".product-item")) > product_count)

    # Update the product count
    product_count = len(driver.find_elements(By.CSS_SELECTOR, ".product-item"))

    # Get the new page height
    new_scroll_height = driver.execute_script("return document.body.scrollHeight")

    # If no new content has been loaded
    if new_scroll_height == scroll_height:
        break

    # Update scroll height and increment scroll count
    scroll_height = new_scroll_height
    scroll_count += 1

# Scrape product details after infinite scrolling
products = []
product_elements = driver.find_elements(By.CSS_SELECTOR, ".product-item")
for product_element in product_elements:
    # Extract product details
    name = product_element.find_element(By.CSS_SELECTOR, ".product-name").text
    image = product_element.find_element(By.CSS_SELECTOR, ".product-image").get_attribute("src")
    price = product_element.find_element(By.CSS_SELECTOR, ".product-price").text
    url = product_element.find_element(By.CSS_SELECTOR, "a").get_attribute("href")

    # Store the scraped data
    products.append({
        "name": name,
        "image": image,
        "price": price,
        "url": url
    })

# Export to CSV/JSON...

# Close the browser driver
driver.quit() 
```

このスクリプトは、まず現在のページ高さとプロダクト数を特定することで無限スクロールを処理します。スクロール処理は最大10回の反復に制限されます。各反復では次を実行します。

1.  最下部までスクロールします
2.  プロダクト数が増えるまで待機します（新しいコンテンツが読み込まれたことを示します）
3.  ページ高さを比較して、追加コンテンツが利用可能かどうかを検出します

スクロール後もページ高さが変わらない場合、ループは終了し、これ以上読み込むデータがないことを示します。

## 結論

複雑なナビゲーションパターンが関与するとWebスクレイピングは困難になり得ます。さらに企業側が、アンチスクレイピング対策を採用して自動化スクリプトをブロックすると、いっそう難しくなります。Selenium のようなブラウザ自動化ツールでは、それらの制限を回避できません。

解決策は、Playwright、Puppeteer、Selenium などのツールと統合でき、各リクエストでIPを自動的にローテーションするクラウドベースのブラウザである [Scraping Browser](https://brightdata.jp/products/scraping-browser) を使用することです。ブラウザフィンガープリント、リトライ、CAPTCHA 解決などを管理できます。複雑なサイトをナビゲートする際にブロックされる状況に別れを告げましょう！