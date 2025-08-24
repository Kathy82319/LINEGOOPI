//【全域設定-金鑰】
const properties = PropertiesService.getScriptProperties();
const FINMIND_API_TOKEN = properties.getProperty('FINMIND_API_TOKEN');


// 當新增一檔股票時，呼叫此函式，即可依序跑完所有更新流程。
function runCompleteUpdateForSingleStock(ticker) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到 "風控報表" 工作表');
    return null;
  }
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  const tickerCol = headers.indexOf('股票代碼');
  let targetRowIndex = -1;

  for (let i = 1; i < data.length; i++) {
    if (data[i][tickerCol] == ticker) {
      targetRowIndex = i;
      break;
    }
  }

  if (targetRowIndex === -1) {
    Logger.log(`❌ 在風控報表中找不到股票 ${ticker}，無法執行單一更新。`);
    return null;
  }

  let rowData = data[targetRowIndex];
  const colMap = headers.reduce((map, header, index) => {
    map[header] = index;
    return map;
  }, {});

  Logger.log(`🚀 開始為 ${ticker} 執行完整的資料初始化流程...`);

  // ★★★ 關鍵修正處：在這裡定義正確的 colMapping 物件 ★★★
  const colMappingForFinancials = {
    '營業收入': ['Revenue'], '營業毛利': ['GrossProfit'], '營業利益': ['OperatingIncome'],
    '稅後淨利': ['IncomeAfterTaxes', 'EquityAttributableToOwnersOfParent'], '營業費用': ['OperatingExpenses'],
    '每股盈餘': ['EPS', 'BasicEarningsPerShare'], '存貨': ['Inventories'],
    '本期綜合損益總額': ['TotalConsolidatedProfitForThePeriod', 'ComprehensiveIncomeConsolidatedNetIncomeAttributedNonControllingInterest'],
    '流動資產總計': ['CurrentAssets'], '非流動資產總計': ['NoncurrentAssets'],
    '流動負債總計': ['CurrentLiabilities'], '非流動負債總計': ['NoncurrentLiabilities'],
    '股東權益總額': ['Equity'], '資產總額': ['TotalAssets'], '負債總額': ['Liabilities']
  };

  // --- 階段一：抓取基礎資料 ---
  Logger.log("--> 階段 1/5: 抓取基礎財報與股本資料...");
  const stockInfo = fetchFinMindStockInfo(ticker);
  const dividendData = fetchAllBaseData_Definitive(ticker);
  const historicals = fetchHistoricalFinancials(ticker, 3);
  // ★★★ 修正：傳入正確的 mapping 物件 ★★★
  const latestFinancials = fetchAndParseMultiSourceFinancials(ticker, colMappingForFinancials); 

  // --- 階段二：填充表格 (基本資料 & 財報數據) ---
  Logger.log("--> 階段 2/5: 填充基本面與財務數據...");
  if (stockInfo) {
    rowData[colMap['股票名稱']] = stockInfo.name;
    rowData[colMap['產業別']] = stockInfo.industry;
  }
  if (dividendData) {
    rowData[colMap['除息日']] = dividendData.ex_dividend_date;
    rowData[colMap['股利發放日']] = dividendData.payment_date;
    rowData[colMap['現金股利']] = dividendData.cash_dividend;
    rowData[colMap['股票股利']] = dividendData.stock_dividend;
    rowData[colMap['股利來源']] = dividendData.dividend_source;
    rowData[colMap['在外流通股數']] = dividendData.shares_outstanding;
  }
  rowData[colMap['連續配息年數']] = calculateConsecutiveDividendYears_Definitive(ticker);
  if (latestFinancials && latestFinancials.latest) {
    const latest = latestFinancials.latest;
    // 使用 mapping 來動態填充，避免寫死一堆欄位
    for (const key in latest) {
        if (colMap[key] !== undefined) {
            rowData[colMap[key]] = latest[key];
        }
    }
    if (latest['資產總額'] > 0) {
      rowData[colMap['負債比']] = ((latest['負債總額'] / latest['資產總額']) * 100).toFixed(2) + '%';
    }
  }
  if (latestFinancials && latestFinancials.latest && latestFinancials.lastYear) {
      const latest = latestFinancials.latest;
      const lastYear = latestFinancials.lastYear;
      if (lastYear['營業收入'] > 0) rowData[colMap['營收 YoY']] = (((latest['營業收入'] - lastYear['營業收入']) / Math.abs(lastYear['營業收入'])) * 100).toFixed(2) + '%';
      if (lastYear['每股盈餘'] != 0) rowData[colMap['EPS YoY']] = (((latest['每股盈餘'] - lastYear['每股盈餘']) / Math.abs(lastYear['每股盈餘'])) * 100).toFixed(2) + '%';
  }
  if (historicals && historicals.length >= 4) {
    const lastFour = historicals.slice(0, 4);
    rowData[colMap['EPS (近四季)']] = lastFour.reduce((sum, q) => sum + (q.EPS || 0), 0).toFixed(2);
    rowData[colMap['營業毛利 (近四季)']] = lastFour.reduce((sum, q) => sum + (q.GrossProfit || 0), 0);
    rowData[colMap['營業收入 (近四季)']] = lastFour.reduce((sum, q) => sum + (q.Revenue || 0), 0);
    rowData[colMap['稅後淨利 (近四季)']] = lastFour.reduce((sum, q) => sum + (q.NetIncome || 0), 0);
  }

  // --- 階段三：抓取即時市場數據 ---
  Logger.log("--> 階段 3/5: 抓取即時市場數據...");
  let latestPriceData = null;
  for (let j = 0; j < 5; j++) {
      let tryDate = new Date();
      tryDate.setDate(tryDate.getDate() - j);
      const dateStr = Utilities.formatDate(tryDate, "Asia/Taipei", "yyyy-MM-dd");
      const result = fetchFinMindStockPrice(ticker, dateStr);
      if (result) { latestPriceData = result; break; }
  }
  const institutionalData = fetchFinMindInstitutionalInvestors(ticker);
  const marginData = fetchFinMindMarginData(ticker);
  const techIndicators = fetchAndCalculateTechIndicators(ticker);

  // --- 階段四：填充即時數據並完成所有衍生計算 ---
  Logger.log("--> 階段 4/5: 填充市場數據並完成衍生計算...");
  if (latestPriceData) {
    const price = latestPriceData.price;
    rowData[colMap['今日股價']] = price;
    rowData[colMap['今日成交量']] = latestPriceData.volume;
    const shares = parseFloat(rowData[colMap['在外流通股數']]);
    const ttmEps = parseFloat(rowData[colMap['EPS (近四季)']]);
    const ttmRevenue = parseFloat(rowData[colMap['營業收入 (近四季)']]);
    const equity = parseFloat(rowData[colMap['股東權益總額']]);
    if (shares > 0) {
        if (ttmRevenue) {
            const sps = ttmRevenue / shares;
            rowData[colMap['每股營收']] = sps.toFixed(2);
            if (sps > 0) rowData[colMap['股價營收比']] = (price / sps).toFixed(2);
        }
        if (equity) {
            const bvps = equity / shares;
            rowData[colMap['每股淨值']] = bvps.toFixed(2);
            if (bvps > 0) rowData[colMap['股價淨值比']] = (price / bvps).toFixed(2);
        }
    }
    if (ttmEps > 0) rowData[colMap['本益比']] = (price / ttmEps).toFixed(2);
    const cashDividend = parseFloat(rowData[colMap['現金股利']]);
    if (cashDividend > 0) {
        const closePriceOnExDate = getPreviousDayClosePrice_Definitive(ticker, rowData[colMap['除息日']]);
        if (closePriceOnExDate > 0) {
            rowData[colMap['殖利率']] = ((cashDividend / closePriceOnExDate) * 100).toFixed(2) + '%';
        }
    }
    if (ttmEps > 0 && cashDividend) rowData[colMap['股利發放率']] = ((cashDividend / ttmEps) * 100).toFixed(2) + '%';
  }
  if (institutionalData) {
    rowData[colMap['外資買超張數']] = institutionalData.foreign_buy_sell;
    rowData[colMap['投信買超張數']] = institutionalData.trust_buy_sell;
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    let historySheet = ss.getSheetByName('法人歷史紀錄');
    if (historySheet) {
      historySheet.appendRow([Utilities.formatDate(new Date(), "Asia/Taipei", "yyyy-MM-dd"), ticker, institutionalData.foreign_buy_sell, institutionalData.trust_buy_sell]);
    }
  }
  if (marginData) {
    rowData[colMap['融資餘額']] = marginData.margin_balance;
    rowData[colMap['券賣餘額']] = marginData.short_balance;
  }
  updateConsecutiveBuyDays_Single(rowData, headers, ticker);
  if (techIndicators) {
    rowData[colMap['近10日均量']] = techIndicators.avgVolume10.toFixed(0);
    const { ma5, ma20, ma60, high60, todayClose, prevClose } = techIndicators;
    if (ma5 > ma20 && ma20 > ma60) rowData[colMap['均線排列']] = '多頭排列';
    else if (ma5 < ma20 && ma20 < ma60) rowData[colMap['均線排列']] = '空頭排列';
    else rowData[colMap['均線排列']] = '盤整';
    rowData[colMap['是否突破前高']] = todayClose >= high60 ? '是' : '否';
    rowData[colMap['是否跌破支撐']] = (prevClose >= ma60 && todayClose < ma60) ? '是' : '否';
  }

  // --- 階段五：抓取並填充非結構化與歷史位階數據 ---
  Logger.log("--> 階段 5/5: 更新新聞情緒與歷史位階...");
  const newsHeadlines = fetchFinMindNews(ticker, 7);
  if (newsHeadlines && newsHeadlines.length > 0) {
    rowData[colMap['近七日新聞則數']] = newsHeadlines.length;
    if(stockInfo && stockInfo.name) {
      rowData[colMap['近期新聞情緒分數']] = analyzeSentimentWithAI(stockInfo.name, ticker, newsHeadlines);
    }
  }
  const currentPE = parseFloat(rowData[colMap['本益比']]);
  if (!isNaN(currentPE)) {
      const historicalPEs = fetchHistoricalPER(ticker, 3);
      if (historicalPEs && historicalPEs.length > 0) {
          let countBelow = 0;
          historicalPEs.forEach(pe => { if (pe < currentPE) countBelow++; });
          rowData[colMap['歷史本益比位階(%)']] = ((countBelow / historicalPEs.length) * 100).toFixed(1) + '%';
      }
  }

  // --- 最終步驟：將更新後的整列數據一次性寫回工作表 ---
  sheet.getRange(targetRowIndex + 1, 1, 1, rowData.length).setValues([rowData]);
  Logger.log(`✅ ${ticker} 的完整資料初始化流程執行完畢！`);
  return rowData;
}

//【每日總開關】負責更新所有每日變動的市場數據、技術指標與籌碼動態。
function runDailyUpdate() {
  const today = new Date();
  const dayOfWeek = today.getDay(); // 獲取今天是星期幾 (0=週日, 1=週一, ..., 6=週六)

  // ★★★ 全新改造：加入「週末守衛」 ★★★
  // 如果今天是週六 (6) 或 週日 (0)，就直接結束函式，不執行任何更新。
  if (dayOfWeek === 6 || dayOfWeek === 0) {
    Logger.log("🚀 [每日] 今天是週末，系統休息中，不執行更新流程。");
    return; // 直接結束函式
  }
  Logger.log("🚀 [每日] 開始執行高頻率更新流程...");
  
  // ★★★ 全新步驟 0: 更新大盤日誌 ★★★
  Logger.log("--> 步驟 0/5: 更新大盤日誌...");
  updateTaiexLog();

  // 順序 1: 【籌碼面指標】：外資買超、投信買超、融資餘額、券賣餘額 
  Logger.log("--> 步驟 1/5: 更新市場數據 (法人、融資券)...");
  updateMarketData();
  
  // 順序 2: 【即時報價】：更新收盤價、本益比、成交量、估值等
  Logger.log("--> 步驟 2/5: 更新股價並計算估值...");
  updateStockPriceAndVolumeFromFinMind();
  
  // 順序 3: 【技術分析模組】：均線、突破跌破等
  Logger.log("--> 步驟 3/5: 計算技術指標...");
  updateTechnicalIndicators();
  
  // 順序 4: 【連買天數計算模組】
  Logger.log("--> 步驟 4/5: 計算法人連買天數...");
  updateConsecutiveBuyDays();

  // 順序 5: 【基本資料模組】：更新股票名稱、產業別
  Logger.log("--> 步驟 5/5: 更新股票基本資料...");
  updateStockInfoFromFinMind();

  // 最終步驟：在所有數據都更新完畢後，執行「動態條件警報」檢查
  Logger.log("--> 最終步驟: 執行「動態條件警報」檢查...");
  checkCustomAlerts();

  Logger.log("✅ [每日] 高頻率更新流程執行完畢！");
}

//【每週總開關】負責更新每週發布的籌碼數據與不常變動的基本資料。
function runWeeklyUpdate() {
  Logger.log("🚀 [每週] 開始執行中頻率更新流程...");

  // 步驟 1: 【股利相關模組】
  Logger.log("--> 步驟 1/3: 更新股利與每股數據...");
  updateDividendModule_Definitive();

  // 步驟 2: 【歷史本益比位階】：
  Logger.log("--> 步驟 2/3: 更新歷史本益比位階...");
  updateHistoricalPERatio();

  // 步驟 3: 【新聞情緒分析】：
  Logger.log("--> 步驟 3/3: 更新新聞數量與 AI 情緒分數...");
  updateNewsSentiment();

  Logger.log("✅ [每週] 中頻率更新流程執行完畢！");
}

//【每季總開關】負責在財報季更新財務報表與滾動歷史數據。
function runQuarterlyUpdate() {
  Logger.log("🚀 [每季] 開始執行財報更新流程...");

  // 順序 1: 【基本面模組】：營業收入、營業毛利、營業利益、稅後淨利、營業費用、每股盈餘、存貨、本期綜合損益總額、流動資產總計、非流動資產總計、流動負債總計、非流動負債總計、股東權益總額、負債比、負債總額、資產總額、營收 YoY、EPS YoY
  Logger.log("--> 步驟 1/2: 更新最新財報...");
  updateFinancials();

  // 順序 2: 【歷史數據滾動模組】EPS (近四季)」、營業毛利 (近四季)、營業收入 (近四季)、稅後淨利 (近四季)
  Logger.log("--> 步驟 2/2: 更新滾動歷史指標 (近四季)...");
  updateHistoricalMetrics();

  Logger.log("✅ [每季] 財報更新流程執行完畢！");
}


//【基本資料模組】：更新股票名稱、產業別
function updateStockInfoFromFinMind() {
  // --- 1. 前置作業：連接工作表並讀取現有資料 ---
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }

  const data = sheet.getDataRange().getValues();
  const headers = data[0];

  // 動態找到所需欄位的索引
  const tickerCol = headers.indexOf('股票代碼');
  const nameCol = headers.indexOf('股票名稱');
  const industryCol = headers.indexOf('產業別');

  if (tickerCol === -1 || nameCol === -1 || industryCol === -1) {
    Logger.log('❌ 找不到必要的欄位：「股票代碼」、「股票名稱」或「產業別」，請檢查標題列。');
    return;
  }

  // --- 2. 核心邏輯：逐一處理每支股票 ---
  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    if (!ticker) continue;

    const stockInfo = fetchFinMindStockInfo(ticker);

    // --- 3. 更新資料：將抓到的資料寫回記憶體中的陣列 ---
    if (stockInfo) {
      Logger.log(`✅ ${ticker}: 找到資料 -> 名稱: ${stockInfo.name}, 產業: ${stockInfo.industry}`);
      data[i][nameCol] = stockInfo.name;
      data[i][industryCol] = stockInfo.industry;
    } else {
      Logger.log(`❌ ${ticker}: 找不到基本資料`);
    }
  }

  // --- 4. 批次寫入：將更新後的整個陣列一次性寫回工作表 ---
  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的基本資料（名稱、產業別）更新完成！');
}

//【基本資料模組輔助函式】：包含 name 和 industry 的物件，或在找不到資料時回傳 null
function fetchFinMindStockInfo(ticker) {
  if (!FINMIND_API_TOKEN) {
    Logger.log("⚠️ FinMind API 金鑰未設定。");
    return null;
  }

  const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockInfo&data_id=${ticker}&token=${FINMIND_API_TOKEN}`;

  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const content = response.getContentText();
    const json = JSON.parse(content);

    if (json.data && json.data.length > 0) {
      // 基本資料通常只有一筆，取第一筆即可
      const info = json.data[0];
      return {
        name: info.stock_name,
        industry: info.industry_category
      };
    } else {
      return null;
    }
  } catch (e) {
    Logger.log(`⚠️ 呼叫 FinMind API 時發生錯誤 (股票: ${ticker}): ${e}`);
    return null;
  }
}

//【即時報價】：更新收盤價、本益比、成交量、券賣佔比、每股淨值比、每股營收比
function updateStockPriceAndVolumeFromFinMind() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];

  // --- 找到所有必要欄位的索引位置 ---
  const tickerCol = headers.indexOf('股票代碼');
  const priceCol = headers.indexOf('今日股價');
  const volumeCol = headers.indexOf('今日成交量');
  
  // 原料
  const sharesOutstandingCol = headers.indexOf('在外流通股數');
  const revenueCol = headers.indexOf('營業收入');
  const ttmEpsCol = headers.indexOf('EPS (近四季)');
  const equityCol = headers.indexOf('股東權益總額');

  // 產出
  const spsCol = headers.indexOf('每股營收');
  const bvpsCol = headers.indexOf('每股淨值');
  
  // 最終估值
  const peRatioCol = headers.indexOf('本益比');
  const psRatioCol = headers.indexOf('股價營收比');
  const pbRatioCol = headers.indexOf('股價淨值比');
  
  if (tickerCol === -1 || priceCol === -1 || volumeCol === -1) {
    Logger.log('❌ 找不到基礎欄位：「股票代碼」、「今日股價」或「今日成交量」。');
    return;
  }

  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    if (!ticker) continue;

    let latestStockData = null;
    let tryDate = new Date();
    for (let j = 0; j < 5; j++) {
      const dateStr = Utilities.formatDate(tryDate, "Asia/Taipei", "yyyy-MM-dd");
      const result = fetchFinMindStockPrice(ticker, dateStr);
      if (result) {
        latestStockData = result;
        break;
      }
      tryDate.setDate(tryDate.getDate() - 1);
    }

    if (latestStockData) {
      const price = latestStockData.price;
      const volume = latestStockData.volume;
      
      data[i][priceCol] = price;
      data[i][volumeCol] = volume;

      const shares = data[i][sharesOutstandingCol];
      let sps = 0; // Sales Per Share (每股營收)
      let bvps = 0; // Book Value Per Share (每股淨值)
      
      if (spsCol !== -1 && revenueCol !== -1 && shares && !isNaN(shares) && shares !== 0) {
        const revenue = data[i][revenueCol];
        if (revenue && !isNaN(revenue)) {
          sps = revenue / shares;
          data[i][spsCol] = sps.toFixed(2);
        } else {
          data[i][spsCol] = '無法計算';
        }
      }
      
      if (bvpsCol !== -1 && equityCol !== -1 && shares && !isNaN(shares) && shares !== 0) {
        const equity = data[i][equityCol];
        if (equity && !isNaN(equity)) {
          bvps = equity / shares;
          data[i][bvpsCol] = bvps.toFixed(2);
        } else {
          data[i][bvpsCol] = '無法計算';
        }
      }

      if (peRatioCol !== -1 && ttmEpsCol !== -1) {
        const ttmEps = data[i][ttmEpsCol];
        if (price > 0 && ttmEps && !isNaN(ttmEps) && ttmEps > 0) {
          data[i][peRatioCol] = (price / ttmEps).toFixed(2);
        } else {
          data[i][peRatioCol] = '無法計算';
        }
      }

      if (pbRatioCol !== -1 && bvps > 0) {
        data[i][pbRatioCol] = (price / bvps).toFixed(2);
      } else if (pbRatioCol !== -1) {
        data[i][pbRatioCol] = '無法計算';
      }

      if (psRatioCol !== -1 && sps > 0) {
        data[i][psRatioCol] = (price / sps).toFixed(2);
      } else if (psRatioCol !== -1) {
        data[i][psRatioCol] = '無法計算';
      }
      
    } else { 
      Logger.log(`❌ ${ticker}: 在過去5天內都找不到股價資料`);
    }
  }

  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的股價、成交量與估值指標更新完成！');
}

//【即時報價輔助函式】包含 price 和 volume 的物件，或在找不到資料時回傳 null
function fetchFinMindStockPrice(ticker, dateStr) {
  if (!FINMIND_API_TOKEN ) {
    Logger.log("⚠️ FinMind API 金鑰未設定，請在程式碼開頭的 FINMIND_API_TOKEN 變數中設定。");
    return null;
  }
  
  const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockPrice&data_id=${ticker}&start_date=${dateStr}&token=${FINMIND_API_TOKEN}`;
  
  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const content = response.getContentText();
    const json = JSON.parse(content);

    if (json.data && json.data.length > 0) {
      // FinMind 可能會回傳多筆，理論上只取最新一筆 (通常是最後一筆)
      const stockInfo = json.data[json.data.length - 1];
      return {
        price: stockInfo.close,
        volume: stockInfo.Trading_Volume
      };
    } else {
      return null;
    }
  } catch (e) {
    Logger.log(`⚠️ 呼叫 FinMind API 時發生錯誤 (股票: ${ticker}, 日期: ${dateStr}): ${e}`);
    return null;
  }
}

//【基本面模組】：營業收入、營業毛利、營業利益、稅後淨利、營業費用、每股盈餘、存貨、本期綜合損益總額、流動資產總計、非流動資產總計、流動負債總計、非流動負債總計、股東權益總額、負債比、負債總額、資產總額、營收 YoY、EPS YoY
function updateFinancials() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }

  const colMapping = {
    '營業收入': ['Revenue'], '營業毛利': ['GrossProfit'], '營業利益': ['OperatingIncome'],
    '稅後淨利': ['IncomeAfterTaxes', 'EquityAttributableToOwnersOfParent'], '營業費用': ['OperatingExpenses'],
    '每股盈餘': ['EPS', 'BasicEarningsPerShare'], '存貨': ['Inventories'],
    '本期綜合損益總額': ['TotalConsolidatedProfitForThePeriod', 'ComprehensiveIncomeConsolidatedNetIncomeAttributedNonControllingInterest'],
    '流動資產總計': ['CurrentAssets'], '非流動資產總計': ['NoncurrentAssets'],
    '流動負債總計': ['CurrentLiabilities'], '非流動負債總計': ['NoncurrentLiabilities'],
    '股東權益總額': ['Equity'], '資產總額': ['TotalAssets'], '負債總額': ['Liabilities']
  };
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  const colIndices = {};
  for (const colName in colMapping) {
    const index = headers.indexOf(colName);
    if (index !== -1) colIndices[colName] = index;
  }
  
  const tickerCol = headers.indexOf('股票代碼');
  const debtRatioCol = headers.indexOf('負債比');
  const revenueYoYCol = headers.indexOf('營收 YoY');
  const epsYoYCol = headers.indexOf('EPS YoY');

  for (let i = 1; i < data.length; i++) {
    // ★★★ 關鍵修正：在讀取股票代碼後，立刻使用 .trim() 去除隱形空格 ★★★
    const ticker = data[i][tickerCol] ? String(data[i][tickerCol]).trim() : null;
    
    if (!ticker) continue;

    const historicalData = fetchAndParseMultiSourceFinancials(ticker, colMapping);

    if (historicalData && historicalData.latest) {
      const financials = historicalData.latest;
      const lastYearFinancials = historicalData.lastYear;
      
      for (const colName in colMapping) {
        const colIndex = colIndices[colName];
        if (colIndex !== -1) {
          data[i][colIndex] = financials[colName] !== undefined ? financials[colName] : '無資料';
        }
      }
      
      if (debtRatioCol !== -1) {
        const totalAssets = financials['資產總額'];
        const totalLiabilities = financials['負債總額'];
        if (totalAssets && !isNaN(totalAssets) && totalLiabilities && !isNaN(totalLiabilities) && totalAssets !== 0) {
          data[i][debtRatioCol] = ((totalLiabilities / totalAssets) * 100).toFixed(2) + '%';
        } else {
          data[i][debtRatioCol] = '無法計算';
        }
      }

      if (revenueYoYCol !== -1) {
        const latestRevenue = financials['營業收入'];
        const lastYearRevenue = lastYearFinancials ? lastYearFinancials['營業收入'] : undefined;
        if (latestRevenue !== undefined && lastYearRevenue !== undefined && lastYearRevenue !== 0) {
          data[i][revenueYoYCol] = (((latestRevenue - lastYearRevenue) / Math.abs(lastYearRevenue)) * 100).toFixed(2) + '%';
        } else {
          data[i][revenueYoYCol] = '資料不足';
        }
      }

      if (epsYoYCol !== -1) {
        const latestEPS = financials['每股盈餘'];
        const lastYearEPS = lastYearFinancials ? lastYearFinancials['每股盈餘'] : undefined;
        if (latestEPS !== undefined && lastYearEPS !== undefined && lastYearEPS !== 0) {
           data[i][epsYoYCol] = (((latestEPS - lastYearEPS) / Math.abs(lastYearEPS)) * 100).toFixed(2) + '%';
        } else {
           data[i][epsYoYCol] = '資料不足';
        }
      }
    }
  }

  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的財務數據及衍生比率更新完成！');
}

//【基本面模組輔助函式】
function fetchAndParseMultiSourceFinancials(ticker, mapping) {
  const datasets = [
    "TaiwanStockFinancialStatements", // 綜合損益表
    "TaiwanStockBalanceSheet"       // 資產負債表
  ];
  let combinedRawData = [];

  // 步驟 1: 遍歷所有需要的資料來源，把原始資料都抓回來
  for (const dataset of datasets) {
    const url = `https://api.finmindtrade.com/api/v4/data?dataset=${dataset}&data_id=${ticker}&start_date=2023-01-01&token=${FINMIND_API_TOKEN}`;
    try {
      const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
      const json = JSON.parse(response.getContentText());
      if (json.data && json.data.length > 0) {
        combinedRawData = combinedRawData.concat(json.data);
      }
    } catch (e) {
      Logger.log(`⚠️ 呼叫 FinMind API 時發生錯誤 (股票: ${ticker}, 資料集: ${dataset}): ${e}`);
    }
  }

  if (combinedRawData.length === 0) {
    Logger.log(`❌ ${ticker}: 找不到任何財報資料。`);
    return null;
  }

  // 步驟 2: 處理合併後的資料，找出所有日期並排序
  const allDates = [...new Set(combinedRawData.map(item => item.date))];
  allDates.sort((a, b) => new Date(b) - new Date(a));
  if (allDates.length === 0) return null;

  // 步驟 3: 像之前一樣，按日期整理資料
  const parsedDataByDate = {};
  for (const date of allDates) {
    const quarterData = combinedRawData.filter(item => item.date === date);
    const results = {};
    for (const colName in mapping) {
      const keywords = mapping[colName];
      const foundItem = quarterData.find(item => keywords.includes(item.type));
      if (foundItem) {
        results[colName] = foundItem.value;
      }
    }
    parsedDataByDate[date] = results;
  }

  const latestDate = allDates[0];
  const lastYearDate = allDates.length >= 5 ? allDates[4] : null; // 假設一年前是 4 季之前

  return {
    latest: parsedDataByDate[latestDate] || null,
    lastYear: lastYearDate ? parsedDataByDate[lastYearDate] : null,
  };
}

//【股利相關】：除息日、股利發放日、現金股利、股票股利、殖利率、股利發放率、填息天數、連續配息年數、在外流通股數、自由現金流、每股自由現金流、股東權益總額、每股營收、每股淨值
function updateDividendModule_Definitive() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }

  // ★★★ 簡化後的欄位對應，只專注於此函式「直接抓取」的數據 ★★★
  const baseDataMapping = {
    '除息日': 'ex_dividend_date',
    '現金股利': 'cash_dividend',
    '股票股利': 'stock_dividend',
    '股利發放日': 'payment_date',
    '在外流通股數': 'shares_outstanding', // 這是本函式最重要的產出之一
    '股利來源': 'dividend_source'
  };
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  const colIndices = {};
  
  // 建立所有欄位的索引，方便後續讀寫
  headers.forEach((header, i) => {
    colIndices[header] = i;
  });
  
  const tickerCol = headers.indexOf('股票代碼');

  // --- 主迴圈：逐一處理股票 ---
  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    if (!ticker) continue;
    
    // 步驟 1: 抓取股利、流通股數等基礎資料
    const baseData = fetchAllBaseData_Definitive(ticker);
    if (baseData) {
      for (const colName in baseDataMapping) {
          if (colIndices[colName] !== undefined) {
              data[i][colIndices[colName]] = baseData[baseDataMapping[colName]] !== undefined ? baseData[baseDataMapping[colName]] : '無';
          }
      }
    }
    
    // 步驟 2: 計算連續配息年數
    const consecutiveYears = calculateConsecutiveDividendYears_Definitive(ticker);
    if (colIndices['連續配息年數'] !== undefined) {
      data[i][colIndices['連續配息年數']] = consecutiveYears;
    }
    
    // 步驟 3: 進行「股利發放率」和「殖利率」等衍生計算
    const cashDividend = data[i][colIndices['現金股利']];
    const ttmEps = data[i][colIndices['EPS (近四季)']]; // 讀取由季報算好的 TTM EPS
    const exDividendDateStr = data[i][colIndices['除息日']];

    // 計算股利發放率
    if (colIndices['股利發放率'] !== undefined) {
      data[i][colIndices['股利發放率']] = (cashDividend && !isNaN(cashDividend) && ttmEps && !isNaN(ttmEps) && ttmEps > 0) ? ((cashDividend / ttmEps) * 100).toFixed(2) + '%' : '無法計算';
    }
    
    // 計算殖利率
    if (colIndices['殖利率'] !== undefined) {
      if (cashDividend > 0 && exDividendDateStr && exDividendDateStr !== '無') {
        const today = new Date();
        const exDividendDate = new Date(exDividendDateStr);
        today.setHours(0, 0, 0, 0); 
        exDividendDate.setHours(0, 0, 0, 0);

        if (exDividendDate > today) {
          data[i][colIndices['殖利率']] = '尚未除息';
        } else {
          const closePrice = getPreviousDayClosePrice_Definitive(ticker, exDividendDateStr);
          if (closePrice > 0) {
            data[i][colIndices['殖利率']] = ((cashDividend / closePrice) * 100).toFixed(2) + '%';
          } else {
            data[i][colIndices['殖利率']] = '無法計算';
          }
        }
      } else {
        data[i][colIndices['殖利率']] = '無配息';
      }
    }
  }
  
  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的股利與每股指標更新完成！');
}

//【股利相關輔助函式】#1：抓取財報 (為了 YoY)
function fetchAndParseHistoricalFinancials(ticker) {
  const colMapping = { '營業收入': ['Revenue'], '每股盈餘': ['EPS', 'BasicEarningsPerShare'], /* ... 其他財報項目 ... */ };
  const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockFinancialStatements&data_id=${ticker}&start_date=2022-01-01&token=${FINMIND_API_TOKEN}`;
  try {
    const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(res.getContentText());
    if (!json.data || json.data.length === 0) { return null; }
    const allDates = [...new Set(json.data.map(item => item.date))].sort((a, b) => new Date(b) - new Date(a));
    if (allDates.length === 0) return null;
    const parsedDataByDate = {};
    for (const date of allDates) {
      const quarterData = json.data.filter(item => item.date === date);
      const results = {};
      for (const colName in colMapping) {
        const keywords = colMapping[colName];
        const foundItem = quarterData.find(item => keywords.includes(item.type));
        if (foundItem) results[colName] = foundItem.value;
      }
      parsedDataByDate[date] = results;
    }
    const latestDate = allDates[0];
    const lastYearDate = allDates.length >= 5 ? allDates[4] : null;
    return { latest: parsedDataByDate[latestDate] || null, lastYear: lastYearDate ? parsedDataByDate[lastYearDate] : null };
  } catch (e) { return null; }
}

//【股利相關輔助函式】#2：抓取股利、流通股數、自由現金流
function fetchAllBaseData_Definitive(ticker) {
  let combinedData = {};

  // 1. 抓取股利資料
  const dividendUrl = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockDividend&data_id=${ticker}&start_date=2023-01-01&token=${FINMIND_API_TOKEN}`;
  try {
    const res = UrlFetchApp.fetch(dividendUrl, { 'muteHttpExceptions': true });
    const json = JSON.parse(res.getContentText());
    if (json.data && json.data.length > 0) {
      const latestDividendData = json.data[json.data.length - 1];
      const cashFromSurplus = latestDividendData.CashStatutorySurplus || 0;
      const dividendSource = cashFromSurplus > 0 ? '含公積金' : '盈餘配發';
      
      Object.assign(combinedData, {
        ex_dividend_date: latestDividendData.CashExDividendTradingDate,
        cash_dividend: latestDividendData.CashEarningsDistribution,
        stock_dividend: latestDividendData.StockEarningsDistribution,
        payment_date: latestDividendData.CashDividendPaymentDate,
        dividend_source: dividendSource
      });
    }
  } catch (e) { /* ... */ }

  // 2. 從「現金流量表」API中，用關鍵字搜尋自由現金流
  const cashFlowUrl = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockCashFlowStatement&data_id=${ticker}&start_date=2023-01-01&token=${FINMIND_API_TOKEN}`;
  try {
    const res = UrlFetchApp.fetch(cashFlowUrl, { 'muteHttpExceptions': true });
    const json = JSON.parse(res.getContentText());
    if (json.data && json.data.length > 0) {
      const latestDate = json.data.reduce((max, p) => (p.date > max ? p.date : max), json.data[0].date);
      const latestData = json.data.filter(item => item.date === latestDate);
      const fcfItem = latestData.find(item => item.type === 'FreeCashFlow');
      if (fcfItem) combinedData['free_cash_flow'] = fcfItem.value;
    }
  } catch (e) { /* ... */ }
  
  // 3. 從「資產負債表」API中，用關鍵字搜尋在外流通股數
  const balanceSheetUrl = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockBalanceSheet&data_id=${ticker}&start_date=2023-01-01&token=${FINMIND_API_TOKEN}`;
  try {
    const res = UrlFetchApp.fetch(balanceSheetUrl, { 'muteHttpExceptions': true });
    const json = JSON.parse(res.getContentText());
     if (json.data && json.data.length > 0) {
      const latestDate = json.data.reduce((max, p) => (p.date > max ? p.date : max), json.data[0].date);
      const latestData = json.data.filter(item => item.date === latestDate);
      const sharesKeywords = ['NumberOfSharesOutstanding', 'OrdinaryShare'];
      const sharesItem = latestData.find(item => sharesKeywords.includes(item.type));
      if (sharesItem) combinedData['shares_outstanding'] = sharesItem.value;
    }
  } catch (e) { /* ... */ }
  
  return combinedData;
}

//【股利相關輔助函式】 #3：計算連續配息年數
function calculateConsecutiveDividendYears_Definitive(ticker) {
    const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockDividend&data_id=${ticker}&start_date=2000-01-01&token=${FINMIND_API_TOKEN}`;
    try {
        const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
        const json = JSON.parse(res.getContentText());
        if (!json.data || json.data.length === 0) return 0;

        const dividendYears = json.data
            .filter(record => record.CashEarningsDistribution > 0)
            .map(record => parseInt(record.year.replace('年', '')) + 1911); 
            
        if (dividendYears.length === 0) return 0;

        const uniqueYears = [...new Set(dividendYears)].sort((a, b) => b - a);
        
        let consecutiveCount = 0;
        if (uniqueYears.length > 0) {
            const latestYear = uniqueYears[0];
            const currentActualYear = new Date().getFullYear();
            if (latestYear < currentActualYear - 1) {
                return 0;
            }
            for (let i = 0; i < uniqueYears.length; i++) {
                if (uniqueYears[i] === latestYear - i) {
                    consecutiveCount++;
                } else {
                    break;
                }
            }
        }
        return consecutiveCount;
    } catch (e) { return 0; }
}

//【股利相關輔助函式】 #4：計算填息天數
function calculateDaysToFillGap(ticker) {
  try {
    const dividendUrl = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockDividend&data_id=${ticker}&start_date=${new Date().getFullYear() - 2}-01-01&token=${FINMIND_API_TOKEN}`;
    const divRes = UrlFetchApp.fetch(dividendUrl, { 'muteHttpExceptions': true });
    const divJson = JSON.parse(divRes.getContentText());
    if (!divJson.data || divJson.data.length === 0) return '無股利資料';
    const recentDividends = divJson.data.filter(d => d.CashEarningsDistribution > 0 && d.CashExDividendTradingDate);
    if (recentDividends.length === 0) return '無現金股利';
    const latestDividend = recentDividends[recentDividends.length - 1];
    const exDividendDate = latestDividend.CashExDividendTradingDate;
    const targetPrice = getPreviousDayClosePrice(ticker, exDividendDate);
    if (!targetPrice) return '無法取得目標價';
    const priceUrl = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockPrice&data_id=${ticker}&start_date=${exDividendDate}&token=${FINMIND_API_TOKEN}`;
    const priceRes = UrlFetchApp.fetch(priceUrl, { 'muteHttpExceptions': true });
    const priceJson = JSON.parse(priceRes.getContentText());
    if (!priceJson.data || priceJson.data.length === 0) return '無法取得股價';
    const maxSearchDays = 252;
    for (let i = 0; i < priceJson.data.length && i < maxSearchDays; i++) {
      if (priceJson.data[i].close >= targetPrice) {
        return i + 1;
      }
    }
    return '未填息';
  } catch (e) { return '計算失敗'; }
}

//【股利相關輔助函式】 #5：取得收盤價
function getPreviousDayClosePrice_Definitive(ticker, dateStr) {
  // ★★★ 修正點 #2：加入偵錯日誌 ★★★
  Logger.log(`[殖利率偵錯] 開始為 ${ticker} 查詢 ${dateStr} 之前的收盤價...`);
  try {
    const targetDate = new Date(dateStr);
    if (isNaN(targetDate.getTime())) {
      Logger.log(`[殖利率偵錯] 失敗：傳入的日期 "${dateStr}" 無效。`);
      return null;
    }
    
    for (let i = 0; i < 5; i++) {
      targetDate.setDate(targetDate.getDate() - 1);
      const queryDate = Utilities.formatDate(targetDate, "Asia/Taipei", "yyyy-MM-dd");
      Logger.log(`[殖利率偵錯] -> 嘗試查詢日期: ${queryDate}`);
      
      const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockPrice&data_id=${ticker}&start_date=${queryDate}&end_date=${queryDate}&token=${FINMIND_API_TOKEN}`;
      const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
      const json = JSON.parse(res.getContentText());

      if (json.data && json.data.length > 0) {
        Logger.log(`[殖利率偵錯] -> 成功找到價格: ${json.data[0].close}`);
        return json.data[0].close;
      } else {
        Logger.log(`[殖利率偵錯] -> 日期 ${queryDate} 查無資料，繼續往前找...`);
      }
    }
    Logger.log(`[殖利率偵錯] 失敗：往前找了5天，都找不到 ${ticker} 的收盤價。`);
    return null;
  } catch (e) {
    Logger.log(`[殖利率偵錯] 失敗：查詢過程中發生程式錯誤: ${e}`);
    return null;
  }
}

//偵錯小工具：檢查指定股票的最新財報中，到底有哪些可用的會計項目
function checkLatestData() {
  const ticker = "6239"; // 👈 在這裡修改你想檢查的股票代碼

  Logger.log(`🔍 開始檢查股票 ${ticker} 的最新財報資料...`);

  const datasets = {
    "綜合財報": "TaiwanStockFinancialStatements",
    "現金流量表": "TaiwanStockCashFlowStatement",
    "資產負債表": "TaiwanStockBalanceSheet"
  };

  for (const [name, dataset] of Object.entries(datasets)) {
    const url = `https://api.finmindtrade.com/api/v4/data?dataset=${dataset}&data_id=${ticker}&start_date=2023-01-01&token=${FINMIND_API_TOKEN}`;
    try {
      const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
      const json = JSON.parse(res.getContentText());
      if (json.data && json.data.length > 0) {
        const latestDate = json.data.reduce((max, p) => (p.date > max ? p.date : max), json.data[0].date);
        const latestData = json.data.filter(item => item.date === latestDate);
        const itemTypes = latestData.map(item => item.type);
        Logger.log(`\n--- ${name} (日期: ${latestDate}) ---\n可用的項目列表: ${JSON.stringify(itemTypes)}\n`);
      } else {
        Logger.log(`\n--- ${name} ---\n找不到任何資料。\n`);
      }
    } catch (e) {
      Logger.log(`\n--- ${name} ---\n查詢失敗: ${e}\n`);
    }
  }
}

//【歷史數據滾動模組】EPS (近四季)」、營業毛利 (近四季)、營業收入 (近四季)、稅後淨利 (近四季)
function updateHistoricalMetrics() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  
  // --- 定義與檢查欄位 ---
  const tickerCol = headers.indexOf('股票代碼');
  // ★ 新增我們要計算的欄位
  const ttmEpsCol = headers.indexOf('EPS (近四季)');
  const ttmGrossProfitCol = headers.indexOf('營業毛利 (近四季)');
  const ttmRevenueCol = headers.indexOf('營業收入 (近四季)');
  const ttmNetIncomeCol = headers.indexOf('稅後淨利 (近四季)');
  
  // 簡單檢查股票代碼欄位是否存在
  if (tickerCol === -1) {
    Logger.log('❌ 執行中止：找不到「股票代碼」欄位。');
    return;
  }

  // --- 主迴圈：逐一處理股票 ---
  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    if (!ticker) continue;
    
    // 步驟 1: 呼叫資料引擎，抓取近3年的歷史財報
    const historicalData = fetchHistoricalFinancials(ticker, 3);
    
    // 步驟 2: 檢查資料是否充足
    if (historicalData && historicalData.length >= 4) {
      const lastFourQuarters = historicalData.slice(0, 4);
      
      // --- 計算「EPS (近四季)」 ---
      if (ttmEpsCol !== -1) {
        const ttmEps = lastFourQuarters.reduce((sum, quarter) => {
          const eps = quarter.EPS;
          return (eps && !isNaN(eps)) ? sum + eps : sum;
        }, 0);
        data[i][ttmEpsCol] = ttmEps.toFixed(2);
      }
      
      // ★★★ 新增：「營業毛利 (近四季)」計算 ★★★
      if (ttmGrossProfitCol !== -1) {
        const ttmGrossProfit = lastFourQuarters.reduce((sum, quarter) => {
          const grossProfit = quarter.GrossProfit;
          return (grossProfit && !isNaN(grossProfit)) ? sum + grossProfit : sum;
        }, 0);
        data[i][ttmGrossProfitCol] = ttmGrossProfit;
      }

      // ★★★ 新增：「營業收入 (近四季)」計算 ★★★
      if (ttmRevenueCol !== -1) {
        const ttmRevenue = lastFourQuarters.reduce((sum, quarter) => {
          const revenue = quarter.Revenue;
          return (revenue && !isNaN(revenue)) ? sum + revenue : sum;
        }, 0);
        data[i][ttmRevenueCol] = ttmRevenue;
      }

      // ★★★ 新增：「稅後淨利 (近四季)」計算 ★★★
      if (ttmNetIncomeCol !== -1) {
        const ttmNetIncome = lastFourQuarters.reduce((sum, quarter) => {
          const netIncome = quarter.NetIncome;
          return (netIncome && !isNaN(netIncome)) ? sum + netIncome : sum;
        }, 0);
        data[i][ttmNetIncomeCol] = ttmNetIncome;
      }
      
      Logger.log(`✅ ${ticker}: 近四季滾動指標計算完成`);
      
    } else {
      // 如果資料不足，將所有相關欄位都標示
      if (ttmEpsCol !== -1) data[i][ttmEpsCol] = '資料不足';
      if (ttmGrossProfitCol !== -1) data[i][ttmGrossProfitCol] = '資料不足';
      if (ttmRevenueCol !== -1) data[i][ttmRevenueCol] = '資料不足';
      if (ttmNetIncomeCol !== -1) data[i][ttmNetIncomeCol] = '資料不足';
      Logger.log(`⚠️ ${ticker}: 歷史財報不足4季，無法計算滾動指標`);
    }
  }

  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的歷史數據指標更新完成！');
}

//【歷史數據滾動模組輔助函式】
function fetchHistoricalFinancials(ticker, years) {
  const startDate = new Date();
  startDate.setFullYear(startDate.getFullYear() - years);
  const startDateStr = Utilities.formatDate(startDate, "Asia/Taipei", "yyyy-MM-dd");

  // ★ 關鍵修正：同時從「綜合損益表」和「現金流量表」等來源獲取數據，讓資料更完整
  const datasets = ["TaiwanStockFinancialStatements", "TaiwanStockCashFlowStatement"];
  let combinedRawData = [];

  for (const dataset of datasets) {
    const url = `https://api.finmindtrade.com/api/v4/data?dataset=${dataset}&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;
    try {
      const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
      const json = JSON.parse(res.getContentText());
      if (json.data && json.data.length > 0) {
        combinedRawData = combinedRawData.concat(json.data);
      }
    } catch (e) {
      Logger.log(`⚠️ 在抓取 ${ticker} 的 ${dataset} 資料時發生錯誤: ${e}`);
    }
  }

  if (combinedRawData.length === 0) return null;

  // --- 後續的資料解析邏輯 (與您現有的版本類似，但更穩健) ---
  const mapping = {
    'Revenue': ['Revenue'], 'GrossProfit': ['GrossProfit'],
    'OperatingIncome': ['OperatingIncome'], 'NetIncome': ['IncomeAfterTaxes', 'EquityAttributableToOwnersOfParent'],
    'EPS': ['EPS', 'BasicEarningsPerShare'], 'FreeCashFlow': ['FreeCashFlow']
  };

  const allDates = [...new Set(combinedRawData.map(item => item.date))].sort((a, b) => new Date(b) - new Date(a));
  if (allDates.length === 0) return null;

  const parsedHistory = [];
  for (const date of allDates) {
    const quarterRawData = combinedRawData.filter(item => item.date === date);
    const quarterResult = { date: date };

    for (const key in mapping) {
      const keywords = mapping[key];
      const foundItem = quarterRawData.find(item => keywords.includes(item.type));
      quarterResult[key] = foundItem ? foundItem.value : null;
    }
    parsedHistory.push(quarterResult);
  }
  
  return parsedHistory;
}

//【籌碼面指標】：外資買超、投信買超、融資餘額、券賣餘額
//在每日更新市場數據後，會自動將最新的「外資」與「投信」買賣超數據，寫入到名為「法人歷史紀錄」的新工作表中。
function updateMarketData() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const reportSheet = ss.getSheetByName('風控報表');
  if (!reportSheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }

  // 步驟 1: 確保「法人歷史紀錄」工作表存在
  let historySheet = ss.getSheetByName('法人歷史紀錄');
  if (!historySheet) {
    historySheet = ss.insertSheet('法人歷史紀錄');
    historySheet.appendRow(['日期', '股票代碼', '外資買超張數', '投信買超張數']);
    Logger.log("✅ 已建立新的工作表: 法人歷史紀錄");
  }
  
  const reportData = reportSheet.getDataRange().getValues();
  const headers = reportData[0];
  
  // --- 找到所有需要更新的籌碼欄位索引 ---
  const tickerCol = headers.indexOf('股票代碼');
  const foreignBuyCol = headers.indexOf('外資買超張數');
  const trustBuyCol = headers.indexOf('投信買超張數');
  const marginBalanceCol = headers.indexOf('融資餘額');
  const shortBalanceCol = headers.indexOf('券賣餘額');
  
  const todayStr = Utilities.formatDate(new Date(), "Asia/Taipei", "yyyy-MM-dd");

  // --- 主迴圈：逐一處理每一支股票 ---
  for (let i = 1; i < reportData.length; i++) {
    const ticker = reportData[i][tickerCol] ? String(reportData[i][tickerCol]).trim() : null;
    if (!ticker) continue;

    // --- 模組 A: 獲取三大法人買賣超 ---
    const institutionalData = fetchFinMindInstitutionalInvestors(ticker);
    if (institutionalData) {
      const foreignBuySell = institutionalData.foreign_buy_sell;
      const trustBuySell = institutionalData.trust_buy_sell;

      // 更新「風控報表」上的今日數據
      if (foreignBuyCol !== -1) reportData[i][foreignBuyCol] = foreignBuySell;
      if (trustBuyCol !== -1) reportData[i][trustBuyCol] = trustBuySell;
      
      // 將今日數據寫入「法人歷史紀錄」工作表 (為計算連買天數使用)
      historySheet.appendRow([todayStr, ticker, foreignBuySell, trustBuySell]);

    } else {
      Logger.log(`-> ${ticker}: 找不到「三大法人」資料。`);
    }

    //模組 B: 獲取融資融券餘額 
    const marginData = fetchFinMindMarginData(ticker);
    if (marginData) {
      if (marginBalanceCol !== -1) reportData[i][marginBalanceCol] = marginData.margin_balance;
      if (shortBalanceCol !== -1) reportData[i][shortBalanceCol] = marginData.short_balance;
    } else {
      Logger.log(`-> ${ticker}: 找不到「融資融券餘額」資料。`);
    }
  }

  // 將所有更新一次性寫回 Google Sheet
  reportSheet.getDataRange().setValues(reportData);
  Logger.log(`✅ 市場籌碼數據更新完成，並已同步寫入歷史紀錄。`);
}


//【融資融券輔助函式】
function fetchFinMindMarginData(ticker) {
  // FinMind 的融資融券資料集名稱是 "TaiwanStockMarginPurchaseShortSale"
  const dataset = "TaiwanStockMarginPurchaseShortSale";
  
  // 我們往前找 7 天的資料，確保能抓到最近一個交易日的數據
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - 7);
  const startDateStr = Utilities.formatDate(startDate, "Asia/Taipei", "yyyy-MM-dd");

  const url = `https://api.finmindtrade.com/api/v4/data?dataset=${dataset}&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;

  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(response.getContentText());

    if (json.data && json.data.length > 0) {
      // API 可能會回傳多筆，我們只需要最新的一筆 (通常是最後一筆)
      const latestData = json.data[json.data.length - 1];
      return {
        margin_balance: latestData.MarginPurchaseTodayBalance, // 融資餘額
        short_balance: latestData.ShortSaleTodayBalance    // 融券餘額
      };
    } else {
      // 如果 API 回應中沒有資料，就回傳 null
      return null;
    }
  } catch (e) {
    Logger.log(`⚠️ 呼叫 FinMind 融資融券 API 時發生錯誤 (股票: ${ticker}): ${e}`);
    return null;
  }
}

//輔助函式：獲取三大法人買賣超
function fetchFinMindInstitutionalInvestors(ticker) {
  const dataset = "TaiwanStockInstitutionalInvestorsBuySell";
  
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - 7);
  const startDateStr = Utilities.formatDate(startDate, "Asia/Taipei", "yyyy-MM-dd");

  const url = `https://api.finmindtrade.com/api/v4/data?dataset=${dataset}&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;

  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(response.getContentText());

    if (json.data && json.data.length > 0) {
      // API 會回傳一個包含多個法人物件的陣列，我們只需要最新日期的資料
      const latestDate = json.data[json.data.length - 1].date;
      const latestDayData = json.data.filter(item => item.date === latestDate);

      // 從最新日期的資料中，分別找出「外資」和「投信」
      const foreignData = latestDayData.find(item => item.name === 'Foreign_Investor');
      const trustData = latestDayData.find(item => item.name === 'Investment_Trust');

      // 安全地計算買賣超，如果找不到該法人資料，就當作 0
      const foreign_buy_sell = foreignData ? (foreignData.buy - foreignData.sell) / 1000 : 0;
      const trust_buy_sell = trustData ? (trustData.buy - trustData.sell) / 1000 : 0;

      return {
        foreign_buy_sell: foreign_buy_sell, // 單位已換算成「張」
        trust_buy_sell: trust_buy_sell      // 單位已換算成「張」
      };
    } else {
      return null;
    }
  } catch (e) {
    Logger.log(`⚠️ 呼叫 FinMind 三大法人 API 時發生錯誤 (股票: ${ticker}): ${e}`);
    return null;
  }
}

//【連買天數計算模組】
function updateConsecutiveBuyDays() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const reportSheet = ss.getSheetByName('風控報表');
  const historySheet = ss.getSheetByName('法人歷史紀錄');

  if (!reportSheet || !historySheet) {
    Logger.log('❌ 找不到「風控報表」或「法人歷史紀錄」工作表。');
    return;
  }

  // --- 1. 一次性讀取所有資料 ---
  const reportData = reportSheet.getDataRange().getValues();
  const historyData = historySheet.getDataRange().getValues();
  const headers = reportData[0];

  const tickerCol = headers.indexOf('股票代碼');
  const foreignStreakCol = headers.indexOf('外資連買天數');
  const trustStreakCol = headers.indexOf('投信連買天數');
  
  // --- 2. 將歷史資料整理成一個 Map，方便快速查找 ---
  const historyMap = {};
  for (let i = 1; i < historyData.length; i++) {
    const row = historyData[i];
    const date = new Date(row[0]);
    const ticker = row[1];
    const foreignBuy = Number(row[2]);
    const trustBuy = Number(row[3]);
    
    if (!historyMap[ticker]) {
      historyMap[ticker] = [];
    }
    historyMap[ticker].push({ date, foreignBuy, trustBuy });
  }

  // --- 3. 遍歷主報表，計算並準備寫入 ---
  for (let i = 1; i < reportData.length; i++) {
    const ticker = reportData[i][tickerCol];
    if (!ticker || !historyMap[ticker]) continue;

    const stockHistory = historyMap[ticker].sort((a, b) => b.date - a.date); // 確保歷史紀錄由新到舊排序

    // 計算外資連買/賣天數
    if (foreignStreakCol !== -1) {
      reportData[i][foreignStreakCol] = calculateStreak(stockHistory, 'foreignBuy');
    }
    
    // 計算投信連買/賣天數
    if (trustStreakCol !== -1) {
      reportData[i][trustStreakCol] = calculateStreak(stockHistory, 'trustBuy');
    }
  }
  
  // --- 4. 批次寫回主報表 ---
  reportSheet.getDataRange().setValues(reportData);
  Logger.log('✅ 外資與投信連買天數計算完成！');
}

//【連買天數計算模組輔助函式】：連續天數 (正數為連買，負數為連賣)
function calculateStreak(sortedHistory, investorType) {
  if (!sortedHistory || sortedHistory.length === 0) return 0;

  const firstDayAction = sortedHistory[0][investorType];
  if (firstDayAction === 0) return 0;
  
  const isBuyingStreak = firstDayAction > 0;
  let streak = 0;

  for (const record of sortedHistory) {
    const currentAction = record[investorType];
    
    if (isBuyingStreak && currentAction > 0) {
      streak++;
    } else if (!isBuyingStreak && currentAction < 0) {
      streak++;
    } else {
      break; // 中斷連續
    }
  }
  
  return isBuyingStreak ? streak : -streak; // 賣超回傳負數
}

//【技術分析模組】：10日均量、均線排列、是否突破前高、是否跌破支撐
function updateTechnicalIndicators() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  
  // --- 定義與檢查欄位 ---
  const tickerCol = headers.indexOf('股票代碼');
  const avgVolumeCol = headers.indexOf('近10日均量');
  const maStateCol = headers.indexOf('均線排列');
  const breakthroughCol = headers.indexOf('是否突破前高'); // ★ 新增欄位
  const breakdownCol = headers.indexOf('是否跌破支撐');   // ★ 新增欄位
  
  if (tickerCol === -1 || avgVolumeCol === -1 || maStateCol === -1 || breakthroughCol === -1 || breakdownCol === -1) {
    Logger.log('❌ 執行中止：找不到必要的技術指標欄位，請檢查標題。');
    return;
  }

  // --- 主迴圈：逐一處理股票 ---
  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    if (!ticker) continue;
    
    // 呼叫升級後的輔助函式，一次取得所有計算結果
    const indicators = fetchAndCalculateTechIndicators(ticker);
    
    if (indicators) {
      // 寫入近10日均量
      data[i][avgVolumeCol] = Math.round(indicators.avgVolume10); 
      
      // 判斷均線排列
      const { ma5, ma20, ma60, high60, prevClose, todayClose } = indicators;
      if (ma5 && ma20 && ma60) {
        if (ma5 > ma20 && ma20 > ma60) {
          data[i][maStateCol] = '多頭排列';
        } else if (ma5 < ma20 && ma20 < ma60) {
          data[i][maStateCol] = '空頭排列';
        } else {
          data[i][maStateCol] = '盤整';
        }
      } else {
        data[i][maStateCol] = '資料不足';
      }

      // ★★★ 新增：判斷是否突破前高 ★★★
      if (high60 && todayClose) {
        if (todayClose >= high60) {
          data[i][breakthroughCol] = '是';
        } else {
          data[i][breakthroughCol] = '否';
        }
      } else {
        data[i][breakthroughCol] = '資料不足';
      }
      
      // ★★★ 新增：判斷是否跌破支撐 ★★★
      if (ma60 && todayClose && prevClose) {
        // 條件：昨天收盤還在季線之上，但今天收盤掉到季線之下
        if (prevClose >= ma60 && todayClose < ma60) {
          data[i][breakdownCol] = '是';
        } else {
          data[i][breakdownCol] = '否';
        }
      } else {
        data[i][breakdownCol] = '資料不足';
      }
      
    } else {
      // 如果資料不足，將所有相關欄位都標示
      data[i][avgVolumeCol] = '資料不足';
      data[i][maStateCol] = '資料不足';
      data[i][breakthroughCol] = '資料不足';
      data[i][breakdownCol] = '資料不足';
    }
  }

  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的技術指標更新完成！');
}

//【技術分析模組輔助函式】包含 avgVolume10, ma5, ma20, ma60, high60, prevClose, todayClose 的物件
function fetchAndCalculateTechIndicators(ticker) {
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - 100);
  const startDateStr = Utilities.formatDate(startDate, "Asia/Taipei", "yyyy-MM-dd");

  const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockPrice&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;
  
  try {
    const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(res.getContentText());
    
    if (!json.data || json.data.length < 60) return null; // 確保至少有60筆資料

    const calculateMA = (data, days) => {
      if (data.length < days) return null;
      const recentData = data.slice(-days);
      const sum = recentData.reduce((acc, record) => acc + record.close, 0);
      return sum / days;
    };

    // 計算均量
    const recentVolumeData = json.data.slice(-10);
    const totalVolume = recentVolumeData.reduce((sum, record) => sum + record.Trading_Volume, 0);
    const avgVolume10 = (totalVolume / 10) / 1000;

    // 計算均線
    const ma5 = calculateMA(json.data, 5);
    const ma20 = calculateMA(json.data, 20);
    const ma60 = calculateMA(json.data, 60);

    // ★★★ 新增：取得今日收盤、昨日收盤、近60日最高價 ★★★
    const todayClose = json.data[json.data.length - 1].close;
    const prevClose = json.data[json.data.length - 2].close;
    
    const last60days = json.data.slice(-60);
    const high60 = last60days.reduce((max, record) => Math.max(max, record.close), 0);
    
    return { avgVolume10, ma5, ma20, ma60, high60, prevClose, todayClose };

  } catch (e) {
    Logger.log(`⚠️ ${ticker}: 抓取歷史股價時發生錯誤: ${e}`);
    return null;
  }
}

//歷史本益比位階分析模組
function updateHistoricalPERatio() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  
  // 找到需要讀取和寫入的欄位
  const tickerCol = headers.indexOf('股票代碼');
  const currentPECol = headers.indexOf('本益比');
  const pePercentileCol = headers.indexOf('歷史本益比位階(%)');

  if (pePercentileCol === -1) {
    Logger.log('❌ 找不到 "歷史本益比位階(%)" 欄位，無法更新。');
    return;
  }

  // 主迴圈：逐一處理每一支股票
  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    const currentPE = parseFloat(data[i][currentPECol]); // 讀取目前的本益比

    // 忽略沒有股票代碼或目前本益比無法計算的股票
    if (!ticker || isNaN(currentPE)) {
      data[i][pePercentileCol] = '資料不足';
      continue;
    }
    
    // 呼叫輔助函式，抓取過去三年的歷史本益比數據
    const historicalPEs = fetchHistoricalPER(ticker, 3);
    
    if (historicalPEs && historicalPEs.length > 0) {
      // 計算百分位
      let countBelow = 0;
      for (const pe of historicalPEs) {
        if (pe < currentPE) {
          countBelow++;
        }
      }
      const percentile = (countBelow / historicalPEs.length) * 100;
      data[i][pePercentileCol] = percentile.toFixed(1) + '%'; // 寫入計算結果，保留一位小數
      Logger.log(`✅ ${ticker}: 目前本益比 ${currentPE}, 歷史位階 ${percentile.toFixed(1)}%`);
    } else {
      data[i][pePercentileCol] = '無歷史資料';
      Logger.log(`-> ${ticker}: 找不到歷史本益比資料。`);
    }
  }

  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的歷史本益比位階更新完畢！');
}

//輔助函式：從 FinMind 獲取指定股票的歷史本益比數據 
function fetchHistoricalPER(ticker, years) {
  const dataset = "TaiwanStockPER";
  
  const startDate = new Date();
  startDate.setFullYear(startDate.getFullYear() - years);
  const startDateStr = Utilities.formatDate(startDate, "Asia/Taipei", "yyyy-MM-dd");

  const url = `https://api.finmindtrade.com/api/v4/data?dataset=${dataset}&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;

  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(response.getContentText());

    if (json.data && json.data.length > 0) {
      // 我們只需要 PER (本益比) 這個欄位的數值
      return json.data.map(item => item.PER);
    } else {
      return null;
    }
  } catch (e) {
    Logger.log(`⚠️ 呼叫 FinMind 歷史本益比 API 時發生錯誤 (股票: ${ticker}): ${e}`);
    return null;
  }
}

//主函式：更新所有股票的近期新聞數量與 AI 情緒分數 ★
function updateNewsSentiment() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  const data = sheet.getDataRange().getValues();
  const headers = data[0];

  // 找到需要讀取和寫入的欄位
  const tickerCol = headers.indexOf('股票代碼');
  const nameCol = headers.indexOf('股票名稱');
  const newsCountCol = headers.indexOf('近七日新聞則數');
  const sentimentScoreCol = headers.indexOf('近期新聞情緒分數');

  if (newsCountCol === -1 || sentimentScoreCol === -1) {
    Logger.log('❌ 找不到 "近七日新聞則數" 或 "近期新聞情緒分數" 欄位，無法更新。');
    return;
  }

  // 主迴圈：逐一處理每一支股票
  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    const name = data[i][nameCol];
    if (!ticker) continue;

    // 步驟 1: 呼叫輔助函式，抓取過去七天的新聞標題
    const newsHeadlines = fetchFinMindNews(ticker, 7);

    if (newsHeadlines && newsHeadlines.length > 0) {
      // 步驟 2: 更新新聞則數
      data[i][newsCountCol] = newsHeadlines.length;
      Logger.log(`✅ ${name}: 找到 ${newsHeadlines.length} 則新聞，準備進行 AI 情緒分析...`);

      // 步驟 3: 呼叫另一個輔助函式，將新聞標題打包送給 AI 進行分析
      const sentimentScore = analyzeSentimentWithAI(name, ticker, newsHeadlines);
      data[i][sentimentScoreCol] = sentimentScore;
      Logger.log(`-> AI 分析完成，情緒分數為: ${sentimentScore}`);

    } else {
      // 如果找不到新聞
      data[i][newsCountCol] = 0;
      data[i][sentimentScoreCol] = '無新聞';
      Logger.log(`-> ${name}: 找不到近期新聞。`);
    }
  }

  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的新聞情緒分析更新完畢！');
}


// 輔助函式 #1：從 FinMind 獲取指定股票的歷史新聞標題 ★
function fetchFinMindNews(ticker, days) {
  const dataset = "TaiwanStockNews";
  
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - days);
  const startDateStr = Utilities.formatDate(startDate, "Asia/Taipei", "yyyy-MM-dd");

  const url = `https://api.finmindtrade.com/api/v4/data?dataset=${dataset}&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;

  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(response.getContentText());

    if (json.data && json.data.length > 0) {
      // 我們只需要新聞的 "title" (標題)
      return json.data.map(item => item.title);
    } else {
      return null;
    }
  } catch (e) {
    Logger.log(`⚠️ 呼叫 FinMind 新聞 API 時發生錯誤 (股票: ${ticker}): ${e}`);
    return null;
  }
}

//輔助函式 #2：將新聞標題送交 OpenAI 進行情緒分析 ★
function analyzeSentimentWithAI(name, ticker, headlines) {
  const properties = PropertiesService.getScriptProperties();
  const perplexityApiKey = properties.getProperty('PERPLEXITY_API_KEY'); // 讀取 Perplexity Key
  if (!perplexityApiKey) return "錯誤：未設定 Perplexity API Key";

  const allHeadlines = headlines.join('\n'); // 將所有標題合併成一個大字串

  const prompt = `
    你是一位專門分析財經新聞情緒的量化分析師。你的任務是讀取我提供的多則新聞標題，然後給出一個精準的、介於 -1.0 (極度負面) 到 +1.0 (極度正面) 之間的情緒分數。

    分析規則：
    - 完全負面或大利空消息（如：營收衰退、財測下修、重大違約）應接近 -1.0。
    - 完全正面或大利多消息（如：營收創歷史新高、接到大訂單、獲利超乎預期）應接近 +1.0。
    - 中性、客觀、或多空消息混雜的新聞，應接近 0.0。
    - 請只專注於新聞標題本身傳達的情緒，不要加入你自己的市場判斷。
    - 你的回答「只能」是一個數字，不要有任何多餘的文字、解釋或開場白。

    請分析以下關於 "${name} (${ticker})" 的新聞標題：
    ---
    ${allHeadlines}
  `;
  
  // 我們直接使用之前寫好的 callperplxity_forGAS 函式
  const aiResponse = callPerplexity_forGAS(prompt, perplexityApiKey);

  // 嘗試將 AI 的回覆轉換為數字
  const score = parseFloat(aiResponse);
  if (!isNaN(score)) {
    return score.toFixed(2); // 回傳保留兩位小數的數字
  } else {
    // 如果 AI 回傳的不是數字，就返回一個錯誤標記
    Logger.log(`-> AI 回傳的格式非數字: ${aiResponse}`);
    return "AI分析失敗";
  }
}

//【每日風控報告模組】
function generateDailyRiskReport() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const reportSheetName = '風控報告';
  const mainSheet = ss.getSheetByName('風控報表');
  if (!mainSheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  
  const data = mainSheet.getDataRange().getValues();
  const headers = data[0];

  const col = {};
  headers.forEach((header, i) => { col[header] = i; });

  let alertMessages = [];

  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const ticker = row[col['股票代碼']];
    const name = row[col['股票名稱']];
    if (!ticker) continue;

    let stockAlerts = [];
    
    // --- 風控條件定義 ---
    if (row[col['是否跌破支撐']] === '是') { stockAlerts.push('跌破支撐(季線)'); }
    if (row[col['均線排列']] === '空頭排列') { stockAlerts.push('均線空頭排列'); }
    
    const foreignBuy = row[col['外資買超張數']];
    const trustBuy = row[col['投信買超張數']];
    if (foreignBuy < 0 && trustBuy < 0) { stockAlerts.push('外資投信同賣'); }
    
    // ★★★ 修正點：將 volume (股) 除以 1000 換算為「張」再進行比較 ★★★
    const volumeInSheets = row[col['今日成交量']] / 1000; 
    const avgVolume = row[col['近10日均量']];
    if (volumeInSheets > avgVolume * 2 && avgVolume > 100) {
      const volumeRatio = (volumeInSheets / avgVolume).toFixed(1);
      stockAlerts.push(`爆量(${volumeRatio}倍)`);
    }

    if (stockAlerts.length > 0) {
      alertMessages.push(`- ${ticker} ${name}: [${stockAlerts.join(', ')}]`);
    }
  }

  // --- 產生最終的報告文字 ---
  const todayStr = Utilities.formatDate(new Date(), "Asia/Taipei", "yyyy-MM-dd");
  let finalReportText = "";

  if (alertMessages.length > 0) {
    finalReportText = "以下股票觸發警示：\n" + alertMessages.join('\n');
  } else {
    finalReportText = "所有監控目標均未觸發風險條件，一切正常。";
  }

  // ★★★ 升級點：將報告寫入「風控報告」工作表並維護三天紀錄 ★★★
  let reportSheet = ss.getSheetByName(reportSheetName);
  if (!reportSheet) {
    reportSheet = ss.insertSheet(reportSheetName, 0); // 插入到最前面
    reportSheet.appendRow(['日期', '報告內容']);
    reportSheet.setColumnWidth(1, 120); // 設定日期欄寬
    reportSheet.setColumnWidth(2, 500); // 設定報告欄寬
  }
  
  const reportData = reportSheet.getDataRange().getValues();
  let dateExists = false;
  // 檢查今天是否已經有報告了
  for (let i = 1; i < reportData.length; i++) {
    const reportDate = Utilities.formatDate(new Date(reportData[i][0]), "Asia/Taipei", "yyyy-MM-dd");
    if (reportDate === todayStr) {
      // 如果有，就更新內容
      reportSheet.getRange(i + 1, 2).setValue(finalReportText);
      dateExists = true;
      break;
    }
  }

  // 如果今天是新的報告，就插入到最上方
  if (!dateExists) {
    reportSheet.insertRowAfter(1); // 在標題列下方插入新的一行
    reportSheet.getRange("A2").setValue(todayStr);
    reportSheet.getRange("B2").setValue(finalReportText).setWrap(true); // 設定自動換行
  }
  
  // 維護3天紀錄：如果報告超過3天(標題+3筆資料=4行)，就刪除最舊的一筆
  if (reportSheet.getLastRow() > 4) {
    reportSheet.deleteRow(reportSheet.getLastRow());
  }

  Logger.log(`✅ 風控報告已更新至「${reportSheetName}」工作表。`);
}

//GPT推送至LINE(把組合好的訊息，交給郵差去寄送)
function pushToLINEforGAS(messageText, lineToken, lineUserId) {
  const apiUrl = "https://api.line.me/v2/bot/message/push";
  const requestBody = { to: lineUserId, messages: [{ type: "text", text: messageText }] };

  const params = {
    method: 'post',
    headers: { 'Authorization': 'Bearer ' + lineToken },
    contentType: 'application/json',
    payload: JSON.stringify(requestBody),
    muteHttpExceptions: true
  };

  Logger.log("準備發送 LINE 推播請求...");
  const response = UrlFetchApp.fetch(apiUrl, params);

  const responseCode = response.getResponseCode();
  const responseBody = response.getContentText();

  Logger.log("------------------------------------");
  Logger.log("LINE API 回應代碼 (Response Code): " + responseCode);
  Logger.log("LINE API 回應內容 (Response Body): " + responseBody);
  Logger.log("------------------------------------");

  if (responseCode !== 200) {
    throw new Error("LINE Push API 請求失敗，請查看上方日誌。");
  }
}

// 輔助函式：為指定股票搜尋最新的3則財經新聞標題
function fetchNewsForStock_forGAS(ticker, name) {
  // 注意：Google Apps Script 沒有內建的 Google 搜尋函式庫。
  // 這裡我們使用一個進階技巧，透過 UrlFetchApp 去呼叫一個客製化的搜尋引擎 API。
  // 你需要先去 https://programmablesearchengine.google.com/ 建立一個免費的搜尋引擎，
  // 並取得你的 Search Engine ID 和 API Key。
  
  const properties = PropertiesService.getScriptProperties();
  const apiKey = properties.getProperty('GOOGLE_SEARCH_API_KEY'); // 需要在屬性中設定
  const searchEngineId = properties.getProperty('GOOGLE_SEARCH_CX');   // 需要在屬性中設定
  
  if (!apiKey || !searchEngineId) {
    Logger.log("警告：未設定 Google Search API Key 或 Search Engine ID，無法抓取新聞。");
    return "新聞功能未啟用。";
  }

  const query = `${ticker} ${name} 財經新聞`;
  const url = `https://www.googleapis.com/customsearch/v1?key=${apiKey}&cx=${searchEngineId}&q=${encodeURIComponent(query)}&num=3`;

  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(response.getContentText());
    
    if (json.items && json.items.length > 0) {
      const titles = json.items.map((item, index) => `${index + 1}. ${item.title}`).join('\n');
      return titles;
    } else {
      return "找不到相關新聞。";
    }
  } catch (e) {
    Logger.log(`抓取新聞時發生錯誤 (股票: ${ticker}): ${e}`);
    return "抓取新聞時發生錯誤。";
  }
}

// =======================================================================
//★★★ 模組一：動態條件警報系統的核心 ★★★
// 在每日數據更新後執行，檢查所有股票是否觸發使用者自訂的警報條件。
// =======================================================================
function checkCustomAlerts() {
  Logger.log("--> 步驟：開始執行「動態條件警報」檢查...");
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  const tickerCol = headers.indexOf('股票代碼');
  const nameCol = headers.indexOf('股票名稱');
  const conditionCol = headers.indexOf('警報條件');

  if (conditionCol === -1) {
    Logger.log('❌ 在工作表中找不到 "警報條件" 欄位，警報系統無法運作。');
    return;
  }
  
  // 逐一檢查每一支股票
  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const userConditions = row[conditionCol];
    
    // 如果條件欄位是空白的，就直接跳過
    if (!userConditions) {
      continue;
    }
    
    // 呼叫我們的解析器來判斷條件是否滿足
    const result = parseAndCheckConditions(userConditions, headers, row);
    
    // 如果所有條件都滿足...
    if (result.isMet) {
      Logger.log(`✅ 觸發警報！股票: ${row[nameCol]}, 條件: ${userConditions}`);
      
      const ticker = row[tickerCol];
      const name = row[nameCol];
      
      // 組合一則清晰的 LINE 通知訊息
      let message = `🔔【動態條件警報】\n`;
      message += `股票: ${ticker} ${name}\n\n`;
      message += `已觸發您設定的條件：\n`;
      message += `\"${userConditions}\"\n\n`;
      message += `詳細觸發數據：\n${result.details}`;
      
      // 發送到 LINE
      const properties = PropertiesService.getScriptProperties();
      const lineChannelToken = properties.getProperty('LINE_CHANNEL_TOKEN');
      const lineUserId = properties.getProperty('LINE_USER_ID');
      if (lineChannelToken && lineUserId) {
        pushToLINEforGAS(message, lineChannelToken, lineUserId);
      }
    }
  }
  Logger.log("✅ 「動態條件警報」檢查完畢！");
}

//智慧大腦：條件解析器
function parseAndCheckConditions(conditionsString, headers, row) {
  // 步驟 1: 先用 "OR" (||) 分隔，拆解成多個「條件組合」
  const conditionGroups = conditionsString.split('||').map(g => g.trim());

  // 步驟 2: 遍歷每一個「條件組合」
  for (const group of conditionGroups) {
    const conditionsInGroup = group.split(';').map(c => c.trim());
    let allConditionsInGroupMet = true; // 假設這個組合內的所有條件都滿足
    let triggerDetails = ""; // 用來記錄這個組合的觸發細節

    // 步驟 3: 在單一組合內，檢查每一個 "AND" 條件
    for (const condition of conditionsInGroup) {
      const match = condition.match(/(.+?)(>=|<=|>|<|=)(.+)/);
      if (!match) continue;

      const fieldName = match[1].trim();
      const operator = match[2].trim();
      const targetValue = match[3].trim();
      
      const colIndex = headers.indexOf(fieldName);
      if (colIndex === -1) {
        allConditionsInGroupMet = false;
        break; 
      }

      let actualValue = row[colIndex];
      let isMet = false;
      const isTargetNumeric = !isNaN(parseFloat(targetValue));
      
      if (isTargetNumeric) {
        actualValue = parseFloat(actualValue);
        let numericTarget = parseFloat(targetValue);
        if (operator === '>') isMet = actualValue > numericTarget;
        else if (operator === '<') isMet = actualValue < numericTarget;
        else if (operator === '>=') isMet = actualValue >= numericTarget;
        else if (operator === '<=') isMet = actualValue <= numericTarget;
        else if (operator === '=') isMet = actualValue == numericTarget;
      } else {
        if (operator === '=') isMet = String(actualValue).trim() == targetValue;
      }
      
      // 將這個條件的觸發細節記錄下來
      triggerDetails += `• ${fieldName}: ${actualValue} (條件: ${operator}${targetValue})\n`;
      
      if (!isMet) {
        allConditionsInGroupMet = false; // 只要有一個條件不滿足...
        break; // ...就立刻判定這個組合失敗，跳到下一個組合
      }
    }

    // 步驟 4: 如果在檢查完一個組合後，它內部的所有條件都還滿足...
    if (allConditionsInGroupMet) {
      // ...那麼就代表 "OR" 邏輯被觸發了！我們不需要再檢查後面的組合了。
      Logger.log(`✅ 條件組合 "${group}" 被觸發！`);
      return { isMet: true, details: triggerDetails }; // 立刻回報成功！
    }
  }

  // 如果遍歷完所有的組合，都沒有任何一個被觸發，才回報失敗。
  return { isMet: false, details: "" };
}

//輔助函式：呼叫 Perplexity 並回傳分析結果
function callPerplexity_forGAS(prompt, apiKey) {
  const apiUrl = "https://api.perplexity.ai/chat/completions";
  
  const requestBody = {
    model: "sonar", 
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: prompt }
    ]
  };

  const params = {
    method: 'post',
    headers: {
      'Authorization': 'Bearer ' + apiKey,
      'Content-Type': 'application/json'
    },
    payload: JSON.stringify(requestBody),
    muteHttpExceptions: true
  };

  Logger.log("準備發送 Perplexity API 請求...");
  const response = UrlFetchApp.fetch(apiUrl, params);
  const responseCode = response.getResponseCode();
  const responseBody = response.getContentText();

  if (responseCode === 200) {
    const jsonResponse = JSON.parse(responseBody);
    return jsonResponse.choices[0].message.content.trim();
  } else {
    // 為了讓日誌更清晰，我們把錯誤訊息也記錄下來
    Logger.log(`Perplexity API 請求失敗！回應代碼: ${responseCode}, 回應內容: ${responseBody}`);
    // 拋出更詳細的錯誤，方便偵錯
    throw new Error(`Perplexity API 請求失敗！ \n\n回應代碼: ${responseCode}\n回應內容: ${responseBody}`);
  }
}

// LINE 互動查詢核心 ★★★
//Webhook 主入口：這是 Google 執行 Web App 的標準進入點。
function doPost(e) {
  if (e === undefined || e.postData === undefined || e.postData.contents === undefined) {
    return ContentService.createTextOutput("Invalid request").setMimeType(ContentService.MimeType.TEXT);
  }
  const events = JSON.parse(e.postData.contents).events;
  events.forEach(function(event) {
    if (event.type === "message" && event.message.type === "text") {
      const userMessage = event.message.text;
      const replyToken = event.replyToken;
      const userId = event.source.userId;
      handleTextMessage(userMessage, replyToken, userId);
    }
  });
  return ContentService.createTextOutput(JSON.stringify({'status':'ok'})).setMimeType(ContentService.MimeType.JSON);
}


// 訊息處理核心 (指令路由器)
function handleTextMessage(message, replyToken, userId) {
  const trimmedMessage = message.trim();

  // --- 指令 1: 分析個股 (最常用) ---
  const analyzeMatch = trimmedMessage.match(/^(分析|查詢)\s*([\w\d\u4e00-\u9fa5]+)$/);
  if (analyzeMatch) {
    const tickerOrName = analyzeMatch[2];
    Logger.log(`接收到分析指令，目標: ${tickerOrName}`);
    replyToLINE(replyToken, `收到請求，正在為您深度分析「${tickerOrName}」，請稍候...`);
    
    // 步驟 1: 正常產生完整報告
    const reportText = generateSingleStockReport(tickerOrName);
    
    // ★ 步驟 2: 使用分隔符將報告切分成陣列
    const reportParts = reportText.split('---###---').map(part => part.trim()).filter(part => part.length > 0);

    // 步驟 3: 將切分好的陣列交給升級後的 push 函式進行分段發送
    pushToLINEforGAS(reportParts, userId);
    return;
  }
  
  // --- 指令 2: 新增股票至風控報表 ---
  const addMatch = trimmedMessage.match(/^(加入|新增)\s*([\w\d]+)$/);
  if (addMatch) {
    const ticker = addMatch[2];
    Logger.log(`接收到新增指令，目標: ${ticker}`);
    replyToLINE(replyToken, `好的，正在將「${ticker}」加入到您的風控報表中...`);
    const resultMessage = addStockToReport(ticker);

    if (resultMessage.startsWith("✅")) {
        pushToLINEforGAS(resultMessage, userId); 
        const analysisReport = generateSingleStockReport(ticker);
        // ★ 同樣對首次分析的報告進行切分
        const reportParts = analysisReport.split('---###---').map(part => part.trim()).filter(part => part.length > 0);
        pushToLINEforGAS(reportParts, userId);
    } else {
        pushToLINEforGAS(resultMessage, userId);
    }
    return;
  }

  // --- 指令 3: AI 財報季智能助理 ---
  const earningsMatch = trimmedMessage.match(/^(速讀|財報)\s*([\w\d]+)(?:\s+(.*))?$/);
  if (earningsMatch) {
    const ticker = earningsMatch[2];
    const period = earningsMatch[3] || "最新一季"; 
    
    Logger.log(`接收到財報速讀指令，目標: ${ticker}, 期間: ${period}`);
    replyToLINE(replyToken, `收到財報速讀請求，正在為您分析「${ticker} ${period}」的財報...`);
    const reportText = generateEarningsReport(ticker, period);
    pushToLINEforGAS(reportText, userId); // 財報報告較短，暫不分段
    return;
  }

  // --- 指令 4: AI 投資組合風控官 ---
  if (trimmedMessage === "組合分析" || trimmedMessage === "風控報告") {
    Logger.log(`接收到投資組合分析指令`);
    replyToLINE(replyToken, `收到投資組合分析請求，正在為您檢查整體持股風險...`);
    const reportText = generatePortfolioReport();
    pushToLINEforGAS(reportText, userId); // 風控報告較短，暫不分段
    return;
  }
  
  // --- 預設回覆 ---
  const defaultReply = "您好！這是一個股票分析助理。\n\n" +
                     "您可以嘗試以下指令：\n" +
                     "• 分析 [股號/名稱]\n" +
                     "• 加入 [股號]\n" +
                     "• 速讀 [股號] (可選填季度)\n" +
                     "• 組合分析";
  replyToLINE(replyToken, defaultReply);
}


//產生單一個股的深度分析報告
function generateSingleStockReport(tickerOrName) {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sheet = ss.getSheetByName('風控報表');
    if (!sheet) return "錯誤：找不到「風控報表」工作表。";
    
    const data = sheet.getDataRange().getValues();
    const headers = data[0];
    const tickerCol = headers.indexOf('股票代碼');
    const nameCol = headers.indexOf('股票名稱');
    let stockDataRow = null;
    let stockInfo = {};
    let stockCode = "";
    let name = "";
    let source = ""; // 標記資料來源

    // 步驟 1: 嘗試從「風控報表」中尋找資料
    for (let i = 1; i < data.length; i++) {
      if (data[i][tickerCol] == tickerOrName || data[i][nameCol] == tickerOrName) {
        stockDataRow = data[i];
        break;
      }
    }

    // 步驟 2: 判斷資料來源
    if (stockDataRow) {
      // --- 狀況 A: 股票在報表中，使用表內現有數據 ---
      source = "來自您的風控報表";
      Logger.log(`在風控報表中找到 ${tickerOrName}，使用表內資料進行分析。`);
      stockCode = stockDataRow[tickerCol];
      name = stockDataRow[nameCol];
      
      const relevantHeaders = [
        '今日股價', '本益比', '股價淨值比', '營收 YoY', 'EPS YoY', '現金股利', 
        '殖利率', '連續配息年數', 'EPS (近四季)', '外資連買天數', '投信連買天數', 
        '歷史本益比位階(%)', '均線排列', '是否突破前高', '是否跌破支撐', '近期新聞情緒分數'
      ];
      relevantHeaders.forEach(header => {
        const index = headers.indexOf(header);
        if(index !== -1){
            stockInfo[header] = stockDataRow[index];
        }
      });

    } else {
      // --- 狀況 B: 股票不在報表中，啟用即時線上抓取模式 ---
      source = "即時線上查詢";
      stockCode = tickerOrName; // 假設使用者輸入的是代碼
      Logger.log(`風控報表中找不到 ${stockCode}，啟用即時線上抓取模式。`);
      
      // ★ 執行即時資料抓取 (這是一個我們需要新增的輔助函式)
      stockInfo = fetchRealTimeStockData(stockCode);
      
      if (!stockInfo || !stockInfo.股票名稱) {
        return `錯誤：無法查詢到股票 "${stockCode}" 的即時資料，請確認股票代碼是否正確。`;
      }
      name = stockInfo.股票名稱;
    }

    // 步驟 3: 組合 Prompt 並呼叫 AI
    const newsTitles = fetchNewsForStock_forGAS(stockCode, name);
    let promptFooter = `\n\n分析完畢。`;
    if (!stockDataRow) {
        promptFooter = `\n\n---
        **操作提示**: 此股票目前不在您的追蹤清單中。若分析後您想將其納入每日追蹤，請直接傳送指令：\n"加入 ${stockCode}"`;
    }

    const prompt = `
      你是一位專注於【中長期價值投資】與【波段操作】的基金經理人。

      請為我深度分析 "${name} (${stockCode})" 這檔個股，你的報告需要像一份給投資委員會的內部決策備忘錄，並包含以下四點：

      1.  **【長期持有價值評估】**:
         * 從「連續配息年數」、「殖利率」、「EPS (近四季)」等數據，評估這家公司是否具備【穩定獲利】的體質，值得長期持有？
         * 它的「歷史本益比位階」目前在哪個區間？這對長期投資者意味著什麼？

      2.  **【中期成長動能分析】**:
          * 結合「營收YoY」、「EPS YoY」與最新的新聞情緒，判斷公司目前是處於【成長加速】、【成長趨緩】還是【衰退】的階段？

      3.  **【波段操作技術面觀察】**:
          * 從「均線排列」、「是否突破前高/跌破支撐」等指標，判斷目前是否為一個【適合進場】的波段操作時機點？還是應該【觀望或減碼】？

      4.  **【市場情緒與動態 (Market Sentiment & Momentum)】**:
          * 綜合「新聞情緒分數」、「法人買賣超」與「成交量變化」，判斷市場當前對這支股票的【短期】情緒是偏向樂觀、悲觀還是中性？這種情緒是否有基本面支撐？    

      5.  **【綜合策略建議】**:
         * 總結以上分析，給我一個明確的投資策略。例如：「建議在 XXX 價位附近開始分批佈局，目標是 XXX，停損點設在季線下方。」
        
      ★★★ 關鍵指令：請在以上 1, 2, 3, 4, 5 每一個要點分析結束後，都必須加上一行獨立的 "---###---" 作為分隔符。★★★
      ---
      [輸入資料]
      資料來源: ${source}
      量化數據: ${JSON.stringify(stockInfo, null, 2)}
      最新新聞標題: ${newsTitles}
      ${promptFooter}
       `;

    const properties = PropertiesService.getScriptProperties();
    const perplexityApiKey = properties.getProperty('PERPLEXITY_API_KEY');
    if (!perplexityApiKey) return "錯誤：未設定 Perplexity API Key。";
    
    return callPerplexity_forGAS(prompt, perplexityApiKey);

  } catch (err) {
    Logger.log(`generateSingleStockReport 發生錯誤: ${err}`);
    return "抱歉，在產生報告時發生內部錯誤，請稍後再試。";
  }
}

//回傳訊息給 LINE (使用 Reply Token)
function replyToLINE(replyToken, messageText) {
  const url = 'https://api.line.me/v2/bot/message/reply';
  const properties = PropertiesService.getScriptProperties();
  const lineToken = properties.getProperty('LINE_CHANNEL_TOKEN');
  const payload = {
    replyToken: replyToken,
    messages: [{ type: 'text', text: messageText }]
  };
  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: { 'Authorization': 'Bearer ' + lineToken },
    payload: JSON.stringify(payload)
  };
  UrlFetchApp.fetch(url, options);
}


//專門用於「推送長訊息」的 Push 函式 (已內建拆分功能) 
function pushToLINEforGAS(messages, userId) {
  const url = "https://api.line.me/v2/bot/message/push";
  const properties = PropertiesService.getScriptProperties();
  const lineToken = properties.getProperty('LINE_CHANNEL_TOKEN');
  
  let messageObjects = [];

  // 步驟 1: 判斷傳入的 messages 是單一字串還是陣列
  if (Array.isArray(messages)) {
    // 如果是陣列，將陣列中的每個字串都轉換成 LINE 的訊息物件格式
    messageObjects = messages.map(text => ({ type: 'text', text: text }));
  } else if (typeof messages === 'string') {
    // 如果是單一字串，就把它放進一個陣列中
    messageObjects = [{ type: 'text', text: messages }];
  } else {
    Logger.log("❌ pushToLINEforGAS 錯誤：傳入的訊息格式不正確 (既不是字串也不是陣列)");
    return;
  }
  
  // LINE API 一次最多只能發送 5 則 push 訊息
  if (messageObjects.length > 5) {
      messageObjects = messageObjects.slice(0, 5);
      Logger.log("⚠️ 訊息超過 5 則，已自動截斷。");
  }

  if (messageObjects.length === 0) {
    Logger.log("ℹ️ 沒有需要發送的訊息。");
    return;
  }

  const payload = {
    to: userId,
    messages: messageObjects
  };

  const options = {
    method: 'post',
    headers: { 'Authorization': 'Bearer ' + lineToken },
    contentType: 'application/json',
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };
  
  const response = UrlFetchApp.fetch(url, options);
  Logger.log("LINE Push API 回應: " + response.getContentText());
}

//每日大盤日誌 
function updateTaiexLog() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheetName = "大盤日誌";
  let sheet = ss.getSheetByName(sheetName);

  // 步驟 1: 如果工作表不存在，就建立它並加入標題
  if (!sheet) {
    sheet = ss.insertSheet(sheetName, 0); // 插入到最前面
    const headers = ["日期", "收盤指數", "漲跌點數", "漲跌幅(%)", "成交金額(億)"];
    sheet.appendRow(headers);
    Logger.log(`✅ 已建立新的工作表: "${sheetName}"`);
  }

  // 步驟 2: 呼叫輔助函式，從 FinMind 獲取最新的大盤數據
  const taiexData = fetchLatestTaiexData();

  if (taiexData) {
    const todayStr = Utilities.formatDate(new Date(taiexData.date), "Asia/Taipei", "yyyy-MM-dd");
    
    // 檢查今天是否已經記錄過，避免重複寫入
    const lastRowData = sheet.getLastRow() > 1 ? sheet.getRange("A" + sheet.getLastRow()).getValue() : null;
    if (lastRowData && Utilities.formatDate(new Date(lastRowData), "Asia/Taipei", "yyyy-MM-dd") === todayStr) {
        Logger.log("ℹ️ 今日大盤數據已記錄，跳過更新。");
        return;
    }

    // 步驟 3: 將新數據寫入到「大盤日誌」的下一列
    const newRow = [
      todayStr,
      taiexData.close,
      taiexData.change,
      taiexData.percentChange,
      taiexData.tradingValue
    ];
    sheet.appendRow(newRow);
    Logger.log(`✅ 已將 ${todayStr} 的大盤數據寫入 "${sheetName}"`);
  } else {
    Logger.log("❌ 無法獲取今日大盤數據。");
  }
}

//輔助函式：獲取最新的加權指數數據
function fetchLatestTaiexData() {
  const dataset = "TaiwanStockPrice";
  const ticker = "TAIEX"; 
  
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - 7);
  const startDateStr = Utilities.formatDate(startDate, "Asia/Taipei", "yyyy-MM-dd");

  const url = `https://api.finmindtrade.com/api/v4/data?dataset=${dataset}&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;

  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(response.getContentText());

    if (json.data && json.data.length > 0) {
      const latestData = json.data[json.data.length - 1];
      
      const close = latestData.close;
      const change = latestData.spread;
      const tradingValue = latestData.Trading_money; // ★ 關鍵修正 #1：使用正確的欄位名稱

      // ★ 關鍵修正 #2：自己動手計算「漲跌幅(%)」 ★
      // 公式：漲跌點數 / (今日收盤 - 漲跌點數) * 100
      let percentChange = 0;
      const previousClose = close - change; // 計算出昨日收盤價
      if (previousClose !== 0) {
        percentChange = (change / previousClose) * 100;
      }

      return {
        date: latestData.date,
        close: close,
        change: change,
        percentChange: percentChange.toFixed(2), // 回傳我們自己算好的漲跌幅
        tradingValue: (tradingValue / 100000000).toFixed(2) // 將單位從「元」換算成「億元」
      };
    } else {
      return null;
    }
  } catch (e) {
    Logger.log(`⚠️ 呼叫 FinMind 加權指數 API 時發生錯誤: ${e}`);
    return null;
  }
}

// 動態新增股票至風控報表
// =======================================================================
// ★★★ 偵錯強化版：動態新增股票 v5.0 (整合詳細錯誤回報) ★★★
// =======================================================================
function addStockToReport(ticker) {
  try {
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
    if (!sheet) return "錯誤：找不到「風控報表」工作表。";

    const data = sheet.getRange("A:A").getValues();
    for (let i = 0; i < data.length; i++) {
      if (data[i][0] == ticker) {
        return `提醒：「${ticker}」已經存在於您的風控報表中，無需重複加入。`;
      }
    }

    sheet.appendRow([ticker]);
    SpreadsheetApp.flush(); 

    // 呼叫超級更新模組
    const updatedRow = runCompleteUpdateForSingleStock(ticker);

    if (updatedRow) {
      return `✅ 已成功將「${ticker}」加入風控報表，並已為您執行首次的完整資料更新！`;
    } else {
      // 這種情況是函式執行完畢但沒有回傳有效數據，代表內部可能有邏輯問題
      return `❌ 已將「${ticker}」加入列表，但在抓取資料時函式未回傳有效數據(可能為null)，請檢查日誌中是否有警告訊息。`;
    }

  } catch (e) {
    // ★★★ 核心改造處 ★★★
    // 當任何錯誤發生時，我們詳細地記錄它，並組合一個清晰的錯誤訊息回傳給使用者
    Logger.log(`==== 新增股票 ${ticker} 時發生嚴重錯誤 ====`);
    Logger.log(`錯誤類型: ${e.name}`);
    Logger.log(`錯誤訊息: ${e.message}`);
    Logger.log(`錯誤堆疊追蹤: \n${e.stack}`);
    Logger.log(`=======================================`);
    
    const errorMessage = `❌ 新增股票「${ticker}」時發生錯誤！\n\n` +
                       `【偵錯情報】:\n${e.message}\n\n` +
                       `這個錯誤通常意味著在抓取或計算某一項數據時失敗了。請根據此訊息檢查您的工作表欄位名稱或相關輔助函式。`;
                       
    return errorMessage;
  }
}

// ★★★ 全新核心引擎：單一個股更新器 ★★★
// 這個函式負責抓取和計算一支股票需要的所有數據，並回傳一個完整的陣列
function runSingleStockUpdate(ticker, sheet) {
  const headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  let rowData = new Array(headers.length).fill(''); // 建立一個和標題一樣長的空白陣列
  
  // 建立一個 map 來快速查找欄位索引
  const colMap = headers.reduce((map, header, index) => {
    map[header] = index;
    return map;
  }, {});

  rowData[colMap['股票代碼']] = ticker;

  // --- 依序執行各模組的數據抓取與計算 ---
  
  // 1. 基本資料
  const stockInfo = fetchFinMindStockInfo(ticker);
  if (stockInfo) {
    rowData[colMap['股票名稱']] = stockInfo.name;
    rowData[colMap['產業別']] = stockInfo.industry;
  }

  // 2. 歷史財報 (滾動四季)
  const historicalData = fetchHistoricalFinancials(ticker, 3);
  if (historicalData && historicalData.length >= 4) {
    const lastFour = historicalData.slice(0, 4);
    rowData[colMap['EPS (近四季)']] = lastFour.reduce((sum, q) => sum + (q.EPS || 0), 0).toFixed(2);
    rowData[colMap['營業收入 (近四季)']] = lastFour.reduce((sum, q) => sum + (q.Revenue || 0), 0);
  }

  // 3. 股利與流通股數
  const dividendData = fetchAllBaseData_Definitive(ticker);
  if (dividendData) {
      rowData[colMap['在外流通股數']] = dividendData.shares_outstanding;
      rowData[colMap['現金股利']] = dividendData.cash_dividend;
      // ... 其他股利相關欄位
  }

  // 4. 最新股價與估值計算
  const priceData = fetchFinMindStockPrice(ticker, Utilities.formatDate(new Date(), "Asia/Taipei", "yyyy-MM-dd"));
  if (priceData) {
    const price = priceData.price;
    const ttmEps = rowData[colMap['EPS (近四季)']];
    rowData[colMap['今日股價']] = price;
    rowData[colMap['今日成交量']] = priceData.volume;
    if (ttmEps > 0) {
      rowData[colMap['本益比']] = (price / ttmEps).toFixed(2);
    }
  }
  
  // 5. 技術指標
  const techData = fetchAndCalculateTechIndicators(ticker);
  if (techData) {
    const { ma5, ma20, ma60, high60, todayClose } = techData;
    if (ma5 > ma20 && ma20 > ma60) rowData[colMap['均線排列']] = '多頭排列';
    else if (ma5 < ma20 && ma20 < ma60) rowData[colMap['均線排列']] = '空頭排列';
    else rowData[colMap['均線排列']] = '盤整';
    rowData[colMap['是否突破前高']] = todayClose >= high60 ? '是' : '否';
  }
  
  return rowData;
}

//AI 財報季智能助理
function generateEarningsReport(ticker, period) {
  Logger.log(`開始為 ${ticker} 產生財報分析報告...`);
  // ★ 關鍵修正：抓取至少 2 年的歷史財報，確保有去年同期數據可比較
  const financials = fetchHistoricalFinancials(ticker, 2); 

  if (!financials || financials.length < 2) {
    return `資料不足：無法從 FinMind 獲取 ${ticker} 足夠的歷史財報數據來進行比較分析。請確認該股票是否為新上市或資料來源有問題。`;
  }

  // 聰明地找出最新一季、上一季、以及去年同期的數據
  const latestQuarter = financials[0];
  const prevQuarter = financials[1];
  // 去年同期通常是 4 季之前
  const lastYearQuarter = financials.length >= 5 ? financials[4] : null; 

  const prompt = `
      你是一位頂尖的產業分析師，專長是從財報中解讀公司的【短期市場新聞】【長期競爭力】與【未來成長趨勢】。
      請依據我提供的財務數據，為 "${ticker}" 撰寫一份專業的財報速讀報告。

      報告需包含三大部分：

    1.  **【核心營運表現】**:
         * 本季的「營收(Revenue)」與「每股盈餘(EPS)」表現如何？與【去年同期】相比，成長動能是增強還是減弱？

    2.  **【獲利能力與體質檢視】**:
        * 公司的「毛利率(GrossProfit)」和「營業利益率(OperatingIncome)」是否有提升？這反映了什麼樣的產業地位或成本控制能力？
        * 潛在風險: 找出 1-2 個最值得警惕的【風險指標】（例如：負債比過高、營收衰退、本益比位於歷史高檔等）

    3.  **質化護城河評估 (Qualitative Moat Assessment)**:
        * 根據最新的新聞標題，推斷並評估該公司可能擁有哪些經濟護城河？（例如：無形資產、成本優勢、網絡效應等）。新聞中是否有任何資訊可能正在【加寬】或【侵蝕】這條護城河？    
        * 綜合「新聞情緒分數」、「法人買賣超」與「成交量變化」，判斷市場當前對這支股票的【短期】情緒是偏向樂觀、悲觀還是中性？這種情緒是否有基本面支撐？

    4.  **【未來展望分析 (Forward-Looking)】**:
        * 綜合評估，你認為這份財報對公司【未來半年的股價走勢】可能帶來什麼正面或負面的影響？投資人應該關注的下一個關鍵點是什麼？

    關鍵指令：請在以上 1, 2, 3, 4每一個要點分析結束後，都必須加上一行獨立的 "---###---" 作為分隔符。
    ---
    [財務數據]
    - 最新一季 (${latestQuarter.date}): ${JSON.stringify(latestQuarter, null, 2)}
    - 上一季 (${prevQuarter.date}): ${JSON.stringify(prevQuarter, null, 2)}
    - 去年同期 (${lastYearQuarter ? lastYearQuarter.date : 'N/A'}): ${JSON.stringify(lastYearQuarter, null, 2)}
    ---
  `;

  const properties = PropertiesService.getScriptProperties();
  const perplexityApiKey = properties.getProperty('PERPLEXITY_API_KEY');
  return callPerplexity_forGAS(prompt, perplexityApiKey);
}

//我的持股工作表：AI 投資組合風控官 
function generatePortfolioReport() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  // 假設您有一個名為「我的持股」的工作表
  const holdingSheet = ss.getSheetByName('我的持股'); 
  const reportSheet = ss.getSheetByName('風控報表');

  if (!holdingSheet) {
    return "錯誤：找不到名為「我的持股」的工作表。\n請建立該工作表，並至少包含 '股票代碼' 和 '持有成本' 兩欄。";
  }
  if (!reportSheet) {
    return "錯誤：找不到「風控報表」工作表。";
  }

  const holdings = holdingSheet.getDataRange().getValues();
  const reportData = reportSheet.getDataRange().getValues();
  const reportHeaders = reportData[0];
  
  // 將風控報表轉換為以股票代碼為 key 的物件，方便快速查找
  const reportMap = reportData.slice(1).reduce((map, row) => {
    const ticker = row[reportHeaders.indexOf('股票代碼')];
    map[ticker] = row;
    return map;
  }, {});

  let portfolioRisks = [];
  // 從第二行開始讀取持股
  for (let i = 1; i < holdings.length; i++) {
    const ticker = holdings[i][0]; // 假設 A 欄是股票代碼
    if (reportMap[ticker]) {
      const stockData = reportMap[ticker];
      let risks = [];
      // 在此定義您關心的風險條件
      if (stockData[reportHeaders.indexOf('是否跌破支撐')] === '是') risks.push('跌破季線');
      if (stockData[reportHeaders.indexOf('均線排列')] === '空頭排列') risks.push('均線空頭');
      if (stockData[reportHeaders.indexOf('外資連買天數')] < 0 && stockData[reportHeaders.indexOf('投信連買天數')] < 0) risks.push('投信外資同賣');
      
      if (risks.length > 0) {
        portfolioRisks.push(`${ticker} ${stockData[reportHeaders.indexOf('股票名稱')]}: ${risks.join(', ')}`);
      }
    }
  }

  if (portfolioRisks.length === 0) {
    return "✅ 您的投資組合目前未觸發任何重大的技術面與籌碼面風險警示，整體狀況良好。";
  }

  const prompt = `
        你是一位資深的投資組合風控顧問，服務的對象是【中長期價值投資者】。
        以下是我目前投資組合中，觸發風險警示的股票清單。請不要給我短線的停損建議，而是從【資產配置】與【長期佈局】的角度提供建議。

        報告需包含兩點：

        1.  **【風險性質評估】**:
           * 這些警示主要是屬於「短期技術面修正」的風險，還是可能影響到「長期持有價值」的結構性風險？

        2.  **【資產配置調整建議】**:
           * 基於這些風險，我是否需要考慮【調整持股比例】？例如，減碼風險較高的標的，轉而加碼基本面穩固的持股？請點名 1-2 檔最需要我重新審視其【在投資組合中佔比】的股票，並說明原因。
           
        ---
        [風險清單]
        ${portfolioRisks.join('\n')}
        ---
          `;
  
  const properties = PropertiesService.getScriptProperties();
  const perplexityApiKey = properties.getProperty('PERPLEXITY_API_KEY');
  return callPerplexity_forGAS(prompt, perplexityApiKey);
}


//輔助函式：即時抓取單一股票的綜合數據
function fetchRealTimeStockData(ticker) {
  let data = {};

  // 1. 獲取基本資料 (名稱、產業)
  const stockInfo = fetchFinMindStockInfo(ticker);
  if (stockInfo) {
    data.股票名稱 = stockInfo.name;
    data.產業別 = stockInfo.industry;
  } else {
    return null; // 如果連基本資料都抓不到，直接返回
  }

  // 2. 獲取最新股價與成交量
  let latestStockData = null;
  let tryDate = new Date();
  for (let j = 0; j < 5; j++) { // 往前找 5 天
      const dateStr = Utilities.formatDate(tryDate, "Asia/Taipei", "yyyy-MM-dd");
      const result = fetchFinMindStockPrice(ticker, dateStr);
      if (result) {
          latestStockData = result;
          break;
      }
      tryDate.setDate(tryDate.getDate() - 1);
  }
  if (latestStockData) {
    data.今日股價 = latestStockData.price;
    data.今日成交量 = latestStockData.volume;
  }

  // 3. 獲取法人買賣超
  const institutionalData = fetchFinMindInstitutionalInvestors(ticker);
  if (institutionalData) {
    data.外資買超張數 = institutionalData.foreign_buy_sell.toFixed(2);
    data.投信買超張數 = institutionalData.trust_buy_sell.toFixed(2);
  }

  // 4. 獲取歷史本益比位階
  const currentPE = parseFloat(data.本益比);
  if (!isNaN(currentPE)) {
      const historicalPEs = fetchHistoricalPER(ticker, 3);
      if (historicalPEs && historicalPEs.length > 0) {
          let countBelow = 0;
          historicalPEs.forEach(pe => { if (pe < currentPE) countBelow++; });
          data.歷史本益比位階 = ((countBelow / historicalPEs.length) * 100).toFixed(1) + '%';
      }
  }
  
  // 5. 獲取技術指標
  const techIndicators = fetchAndCalculateTechIndicators(ticker);
  if (techIndicators) {
      const { ma5, ma20, ma60, high60, todayClose, prevClose } = techIndicators;
      if (ma5 > ma20 && ma20 > ma60) data.均線排列 = '多頭排列';
      else if (ma5 < ma20 && ma20 < ma60) data.均線排列 = '空頭排列';
      else data.均線排列 = '盤整';
      
      if (todayClose >= high60) data.是否突破前高 = '是'; else data.是否突破前高 = '否';
      if (prevClose >= ma60 && todayClose < ma60) data.是否跌破支撐 = '是'; else data.是否跌破支撐 = '否';
  }

  Logger.log(`即時抓取 ${ticker} 資料完成: ${JSON.stringify(data)}`);
  return data;
}

// =======================================================================
// ★★★ 以下為各模組的「單一股票」執行版本 ★★★
// =======================================================================

function updateMarketData_Single(rowData, headers, ticker) {
  const colMap = headers.reduce((map, header, index) => { map[header] = index; return map; }, {});
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let historySheet = ss.getSheetByName('法人歷史紀錄');
  if (!historySheet) { /* ... 建立表頭 ... */ }
  const todayStr = Utilities.formatDate(new Date(), "Asia/Taipei", "yyyy-MM-dd");

  const institutionalData = fetchFinMindInstitutionalInvestors(ticker);
  if (institutionalData) {
    rowData[colMap['外資買超張數']] = institutionalData.foreign_buy_sell;
    rowData[colMap['投信買超張數']] = institutionalData.trust_buy_sell;
    historySheet.appendRow([todayStr, ticker, institutionalData.foreign_buy_sell, institutionalData.trust_buy_sell]);
  }

  const marginData = fetchFinMindMarginData(ticker);
  if (marginData) {
    rowData[colMap['融資餘額']] = marginData.margin_balance;
    rowData[colMap['券賣餘額']] = marginData.short_balance;
  }
}

function updateStockPrice_Single(rowData, headers, ticker) {
  const colMap = headers.reduce((map, header, index) => { map[header] = index; return map; }, {});
  const latestPriceData = fetchFinMindStockPrice(ticker, Utilities.formatDate(new Date(), "Asia/Taipei", "yyyy-MM-dd"));
  if (latestPriceData) {
    const price = latestPriceData.price;
    const ttmEps = parseFloat(rowData[colMap['EPS (近四季)']]);
    const shares = parseFloat(rowData[colMap['在外流通股數)']]);
    // ... 此處省略完整的估值計算，您可以從 updateStockPriceAndVolumeFromFinMind 函式中複製過來 ...
    rowData[colMap['今日股價']] = price;
    if (ttmEps > 0) rowData[colMap['本益比']] = (price / ttmEps).toFixed(2);
  }
}

function updateTechnicalIndicators_Single(rowData, headers, ticker) {
  const colMap = headers.reduce((map, header, index) => { map[header] = index; return map; }, {});
  const indicators = fetchAndCalculateTechIndicators(ticker);
  if (indicators) {
    const { ma5, ma20, ma60, high60, todayClose, prevClose } = indicators;
    if (ma5 > ma20 && ma20 > ma60) rowData[colMap['均線排列']] = '多頭排列';
    else if (ma5 < ma20 && ma20 < ma60) rowData[colMap['均線排列']] = '空頭排列';
    else rowData[colMap['均線排列']] = '盤整';
    rowData[colMap['是否突破前高']] = todayClose >= high60 ? '是' : '否';
    rowData[colMap['是否跌破支撐']] = (prevClose >= ma60 && todayClose < ma60) ? '是' : '否';
  }
}

function updateConsecutiveBuyDays_Single(rowData, headers, ticker) {
  const colMap = headers.reduce((map, header, index) => { map[header] = index; return map; }, {});
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const historySheet = ss.getSheetByName('法人歷史紀錄');
  if (!historySheet) return;

  const historyData = historySheet.getDataRange().getValues();
  const stockHistory = [];
  for (let i = historyData.length - 1; i >= 1; i--) { // 從後往前找，效率更高
    if (historyData[i][1] == ticker) { // 欄位1是股票代碼
      stockHistory.push({
        date: new Date(historyData[i][0]),
        foreignBuy: Number(historyData[i][2]),
        trustBuy: Number(historyData[i][3])
      });
    }
    if (stockHistory.length > 30) break; // 最多回溯30筆紀錄即可
  }
  
  rowData[colMap['外資連買天數']] = calculateStreak(stockHistory, 'foreignBuy');
  rowData[colMap['投信連買天數']] = calculateStreak(stockHistory, 'trustBuy');
}




















