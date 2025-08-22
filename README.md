//【全域設定-金鑰】
const FINMIND_API_TOKEN = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJkYXRlIjoiMjAyNS0wOC0xNyAwMTo1MjoyOCIsInVzZXJfaWQiOiJjYXRoODIzMTlAZ21haWwuY29tIiwiaXAiOiIxMTQuNDYuMTMzLjEzIn0.5id3AZqo5220p858xB6WKP6RTvc_G3D7R5N-wu8frA4";


//【每日總開關】負責更新所有每日變動的市場數據、技術指標與籌碼動態。
function runDailyUpdate() {
  Logger.log("🚀 [每日] 開始執行高頻率更新流程...");
  
  // 順序 1: 【籌碼面指標】：外資買超、投信買超、融資餘額、券賣餘額、借券餘額、當日券賣
  Logger.log("--> 步驟 1/4: 更新市場數據 (法人、融資券)...");
  updateMarketData();
  
  // 順序 2: 【即時報價】：更新收盤價、本益比、成交量、券賣佔比、每股淨值比、每股營收比
  Logger.log("--> 步驟 2/4: 更新股價並計算估值...");
  updateStockPriceAndVolumeFromFinMind();
  
  // 順序 3: 【技術分析模組】：10日均量、均線排列、是否突破前高、是否跌破支撐
  Logger.log("--> 步驟 3/4: 計算技術指標 (均線、突破跌破)...");
  updateTechnicalIndicators();
  
  // 順序 4: 【連買天數計算模組】
  Logger.log("--> 步驟 4/4: 計算法人連買天數...");
  updateConsecutiveBuyDays();
  
 // ★★★ 最終步驟：在所有數據都更新完畢後，產生風控報告 ★★★
  Logger.log("--> 最終步驟: 產生每日風控報告...");
  generateDailyRiskReport();

  // ★★★ 全新最終步驟：產生 AI 分析報告並推播至 LINE ★★★
  Logger.log("--> 全新最終步驟: 產生 AI 分析報告並推播至 LINE...");
  generateAIReportAndPushToLINE(); 

  Logger.log("✅ [每日] 高頻率更新流程執行完畢！");
}


//【每週總開關】負責更新每週發布的籌碼數據與不常變動的基本資料。
function runWeeklyUpdate() {
  Logger.log("🚀 [每週] 開始執行中頻率更新流程...");

  // 步驟 1: 【大戶集中度模組】
  Logger.log("--> 步驟 1/3: 更新大戶集中度...");
  updateShareholderConcentration_Debug(); // 維持你目前的函式名稱

  // 步驟 2: 【股利相關】：除息日、股利發放日、現金股利、股票股利、殖利率、股利發放率、填息天數、連續配息年數、在外流通股數、自由現金流、每股自由現金流、股東權益總額、每股營收、每股淨值
  Logger.log("--> 步驟 2/3: 更新股利與每股數據...");
  updateDividendModule_Definitive();

  // 步驟 3: 【基本資料模組】：更新股票名稱、產業別
  Logger.log("--> 步驟 3/3: 更新股票基本資料...");
  updateStockInfoFromFinMind();

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
  
  // 計算本益比所需欄位
  const ttmEpsCol = headers.indexOf('EPS (近四季)');
  const peRatioCol = headers.indexOf('本益比');
  
  // ★ 新增：計算股價淨值比所需欄位
  const bvpsCol = headers.indexOf('每股淨值');
  const pbRatioCol = headers.indexOf('股價淨值比');

  // ★ 新增：計算股價營收比所需欄位
  const spsCol = headers.indexOf('每股營收');
  const psRatioCol = headers.indexOf('股價營收比');
  
  if (tickerCol === -1 || priceCol === -1 || volumeCol === -1) {
    Logger.log('❌ 找不到基礎欄位：「股票代碼」、「今日股價」或「今日成交量」。');
    return;
  }

  // --- 主迴圈：逐一處理股票 ---
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
      
      // 寫入股價與成交量
      data[i][priceCol] = price;
      data[i][volumeCol] = volume;

      // --- 進行所有估值計算 ---

      // 計算本益比 (P/E Ratio)
      if (peRatioCol !== -1 && ttmEpsCol !== -1) {
        const ttmEps = data[i][ttmEpsCol];
        if (price > 0 && ttmEps && !isNaN(ttmEps) && ttmEps > 0) {
          data[i][peRatioCol] = (price / ttmEps).toFixed(2);
        } else {
          data[i][peRatioCol] = '無法計算';
        }
      }

      // ★ 新增：計算股價淨值比 (P/B Ratio) ★
      if (pbRatioCol !== -1 && bvpsCol !== -1) {
        const bvps = data[i][bvpsCol]; // 讀取由「股利模組」算好的每股淨值
        if (price > 0 && bvps && !isNaN(bvps) && bvps > 0) {
          data[i][pbRatioCol] = (price / bvps).toFixed(2);
        } else {
          data[i][pbRatioCol] = '無法計算';
        }
      }

      // ★ 新增：計算股價營收比 (P/S Ratio) ★
      if (psRatioCol !== -1 && spsCol !== -1) {
        const sps = data[i][spsCol]; // 讀取由「股利模組」算好的每股營收
        if (price > 0 && sps && !isNaN(sps) && sps > 0) {
          data[i][psRatioCol] = (price / sps).toFixed(2);
        } else {
          data[i][psRatioCol] = '無法計算';
        }
      }
      
    } else { 
      Logger.log(`❌ ${ticker}: 在過去5天內都找不到股價資料`);
    }
  }

  // --- 批次寫入 ---
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

  // 最終版的欄位關鍵字對應
  const colMapping = {
    '營業收入': ['Revenue'],
    '營業毛利': ['GrossProfit'],
    '營業利益': ['OperatingIncome'],
    '稅後淨利': ['IncomeAfterTaxes', 'EquityAttributableToOwnersOfParent'],
    '營業費用': ['OperatingExpenses'],
    '每股盈餘': ['EPS', 'BasicEarningsPerShare'], // 保留多個可能的名稱
    '存貨': ['Inventories'],
    '本期綜合損益總額': ['TotalConsolidatedProfitForThePeriod', 'TotalComprehensiveIncome'],
    '流動資產總計': ['TotalCurrentAssets'],
    '非流動資產總計': ['TotalNonCurrentAssets'],
    '流動負債總計': ['TotalCurrentLiabilities'],
    '非流動負債總計': ['TotalNonCurrentLiabilities'],
    '股東權益總額': ['TotalEquity'],
    '資產總額': ['TotalAssets'],
    '負債總額': ['TotalLiabilities']
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
    const ticker = data[i][tickerCol];
    if (!ticker) continue;

    const historicalData = fetchAndParseHistoricalFinancials(ticker, colMapping);

    if (historicalData && historicalData.latest) {
      const financials = historicalData.latest;
      const lastYearFinancials = historicalData.lastYear;
      
      // 寫入當期數據
      for (const colName in colMapping) {
        const colIndex = colIndices[colName];
        if (colIndex !== -1) {
          data[i][colIndex] = financials[colName] !== undefined ? financials[colName] : '無';
        }
      }
      
      // 計算負債比
      if (debtRatioCol !== -1) {
        const totalAssets = financials['資產總額'];
        const totalLiabilities = financials['負債總額'];
        if (totalAssets && totalLiabilities && totalAssets !== 0) {
          const debtRatio = (totalLiabilities / totalAssets) * 100;
          data[i][debtRatioCol] = debtRatio.toFixed(2) + '%';
        } else {
          data[i][debtRatioCol] = '無法計算';
        }
      }

      // 計算營收 YoY
      if (revenueYoYCol !== -1) {
        const latestRevenue = financials['營業收入'];
        const lastYearRevenue = lastYearFinancials ? lastYearFinancials['營業收入'] : undefined;
        if (latestRevenue !== undefined && lastYearRevenue !== undefined && lastYearRevenue !== 0) {
          const yoy = ((latestRevenue - lastYearRevenue) / Math.abs(lastYearRevenue)) * 100;
          data[i][revenueYoYCol] = yoy.toFixed(2) + '%';
        } else {
          data[i][revenueYoYCol] = '資料不足';
        }
      }

      // 計算 EPS YoY
      if (epsYoYCol !== -1) {
        const latestEPS = financials['每股盈餘'];
        const lastYearEPS = lastYearFinancials ? lastYearFinancials['每股盈餘'] : undefined;
        if (latestEPS !== undefined && lastYearEPS !== undefined && lastYearEPS !== 0) {
           const yoy = ((latestEPS - lastYearEPS) / Math.abs(lastYearEPS)) * 100;
           data[i][epsYoYCol] = yoy.toFixed(2) + '%';
        } else {
           data[i][epsYoYCol] = '資料不足';
        }
      }
    } else {
        // 清空所有相關欄位
    }
  }

  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的財務數據及衍生比率更新完成！');
}

//【基本面模組輔助函式】
function fetchAndParseHistoricalFinancials(ticker, mapping) {
  const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockFinancialStatements&data_id=${ticker}&start_date=2022-01-01&token=${FINMIND_API_TOKEN}`;

  try {
    const response = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(response.getContentText());
    if (!json.data || json.data.length === 0) { return null; }

    const allDates = [...new Set(json.data.map(item => item.date))];
    allDates.sort((a, b) => new Date(b) - new Date(a));

    if (allDates.length === 0) return null;

    const parsedDataByDate = {};
    for (const date of allDates) {
      const quarterData = json.data.filter(item => item.date === date);
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
    const lastYearDate = allDates.length >= 5 ? allDates[4] : null;

    return {
      latest: parsedDataByDate[latestDate] || null,
      lastYear: lastYearDate ? parsedDataByDate[lastYearDate] : null,
    };

  } catch (e) {
    Logger.log(`⚠️ 呼叫 FinMind 財報 API 時發生錯誤 (股票: ${ticker}): ${e}`);
    return null;
  }
}

//【股利相關】：除息日、股利發放日、現金股利、股票股利、殖利率、股利發放率、填息天數、連續配息年數、在外流通股數、自由現金流、每股自由現金流、股東權益總額、每股營收、每股淨值
function updateDividendModule_Definitive() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }

  const baseDataMapping = { '除息日': 'ex_dividend_date', '現金股利': 'cash_dividend', '股票股利': 'stock_dividend', '股利發放日': 'payment_date', '在外流通股數': 'shares_outstanding', '自由現金流': 'free_cash_flow', '股利來源': 'dividend_source' };
  const existingFinancialCols = ['每股盈餘', '股東權益總額', '營業收入', 'EPS (近四季)'];
  const calculatedCols = ['殖利率', '股利發放率', '每股淨值', '每股營收', '每股自由現金流', '連續配息年數'];
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  const colIndices = {};
  [...Object.keys(baseDataMapping), ...existingFinancialCols, ...calculatedCols].forEach(colName => {
    const index = headers.indexOf(colName);
    if (index !== -1) colIndices[colName] = index;
  });
  
  const tickerCol = headers.indexOf('股票代碼');

  // --- 主迴圈：逐一處理股票 ---
  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    if (!ticker) continue;
    
    const baseData = fetchAllBaseData_Definitive(ticker);
    if (baseData) {
      for (const colName in baseDataMapping) {
          if (colIndices[colName] !== undefined) {
              data[i][colIndices[colName]] = baseData[baseDataMapping[colName]] !== undefined ? baseData[baseDataMapping[colName]] : '無';
          }
      }
    }
    const consecutiveYears = calculateConsecutiveDividendYears_Definitive(ticker);
    if (colIndices['連續配息年數'] !== undefined) {
      data[i][colIndices['連續配息年數']] = consecutiveYears;
    }
    
    // 步驟 3: 進行所有衍生計算
    const shares = data[i][colIndices['在外流通股數']];
    const fcf = data[i][colIndices['自由現金流']];
    const equity = data[i][colIndices['股東權益總額']];
    const revenue = data[i][colIndices['營業收入']];
    const cashDividend = data[i][colIndices['現金股利']];
    const ttmEps = data[i][colIndices['EPS (近四季)']]; 
    const exDividendDateStr = data[i][colIndices['除息日']];

    // ... (其他計算邏輯不變) ...
    if (colIndices['每股淨值'] !== undefined) { data[i][colIndices['每股淨值']] = (equity && !isNaN(equity) && shares && !isNaN(shares) && shares !== 0) ? (equity / shares).toFixed(2) : '無法計算'; }
    if (colIndices['每股營收'] !== undefined) { data[i][colIndices['每股營收']] = (revenue && !isNaN(revenue) && shares && !isNaN(shares) && shares !== 0) ? (revenue / shares).toFixed(2) : '無法計算'; }
    if (colIndices['每股自由現金流'] !== undefined) { data[i][colIndices['每股自由現金流']] = (fcf && !isNaN(fcf) && shares && !isNaN(shares) && shares !== 0) ? (fcf / shares).toFixed(2) : '無法計算'; }
    if (colIndices['股利發放率'] !== undefined) { data[i][colIndices['股利發放率']] = (cashDividend && !isNaN(cashDividend) && ttmEps && !isNaN(ttmEps) && ttmEps > 0) ? ((cashDividend / ttmEps) * 100).toFixed(2) + '%' : '無法計算'; }
    
    // ★★★ 優化後的殖利率計算邏輯 ★★★
    if (colIndices['殖利率'] !== undefined) {
      if (cashDividend > 0 && exDividendDateStr && exDividendDateStr !== '無') {
        const today = new Date();
        const exDividendDate = new Date(exDividendDateStr);
        
        // 設定時區為午夜，避免時差問題
        today.setHours(0, 0, 0, 0); 
        exDividendDate.setHours(0, 0, 0, 0);

        if (exDividendDate > today) {
          // 如果除息日是未來
          data[i][colIndices['殖利率']] = '尚未除息';
        } else {
          // 如果除息日是今天或過去，才進行計算
          const closePrice = getPreviousDayClosePrice_Definitive(ticker, exDividendDateStr);
          if (closePrice > 0) {
            const yieldRatio = (cashDividend / closePrice) * 100;
            data[i][colIndices['殖利率']] = yieldRatio.toFixed(2) + '%';
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
  const ticker = "2330"; // 👈 在這裡修改你想檢查的股票代碼

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

  const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockFinancialStatements&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;
  
  try {
    const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(res.getContentText());
    if (!json.data || json.data.length === 0) { return null; }

    const mapping = {
      'Revenue': ['Revenue'],
      'GrossProfit': ['GrossProfit'],
      'OperatingIncome': ['OperatingIncome'],
      'NetIncome': ['IncomeAfterTaxes', 'EquityAttributableToOwnersOfParent'],
      'EPS': ['EPS', 'BasicEarningsPerShare'],
    };

    const allDates = [...new Set(json.data.map(item => item.date))].sort((a, b) => new Date(b) - new Date(a));
    if (allDates.length === 0) return null;

    const parsedHistory = [];
    for (const date of allDates) {
      const quarterRawData = json.data.filter(item => item.date === date);
      const quarterResult = { date: date };

      for (const key in mapping) {
        const keywords = mapping[key];
        const foundItem = quarterRawData.find(item => keywords.includes(item.type));
        quarterResult[key] = foundItem ? foundItem.value : null;
      }
      parsedHistory.push(quarterResult);
    }
    
    return parsedHistory;

  } catch (e) {
    Logger.log(`⚠️ ${ticker}: 抓取歷史財報時發生錯誤: ${e}`);
    return null;
  }
}

//【籌碼面指標】：外資買超、投信買超、融資餘額、券賣餘額、借券餘額、當日券賣
//在每日更新市場數據後，會自動將最新的「外資」與「投信」買賣超數據，寫入到名為「法人歷史紀錄」的新工作表中。
function updateMarketData() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  
  // 呼叫輔助函式，一次性抓回全市場的法人與融資券資料
  const marketDataMap = fetchLatestMarketData();
  
  if (!marketDataMap) {
    Logger.log('❌ 無法從 API 獲取任何市場數據，執行中止。');
    return;
  }

  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  
  const tickerCol = headers.indexOf('股票代碼');
  const foreignBuyCol = headers.indexOf('外資買超張數');
  const trustBuyCol = headers.indexOf('投信買超張數');
  const marginBalanceCol = headers.indexOf('融資餘額');
  const shortBalanceCol = headers.indexOf('券賣餘額');
  const sblBalanceCol = headers.indexOf('借券餘額');
  const shortSaleTodayCol = headers.indexOf('當日券賣');
  
  let newHistoryRows = []; // 用來存放要寫入歷史紀錄的資料
  const dataDate = marketDataMap.data_date.replace(/(\d{4})(\d{2})(\d{2})/, '$1-$2-$3'); // 將 20250815 轉為 2025-08-15

  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    if (!ticker) continue;
    
    const stockMarketData = marketDataMap[ticker];
    
    if (stockMarketData) {
      const foreignNetBuy = stockMarketData.foreign_net_buy || 0;
      const trustNetBuy = stockMarketData.trust_net_buy || 0;
      
      if (foreignBuyCol !== -1) data[i][foreignBuyCol] = foreignNetBuy;
      if (trustBuyCol !== -1) data[i][trustBuyCol] = trustNetBuy;
      if (marginBalanceCol !== -1) data[i][marginBalanceCol] = stockMarketData.margin_balance || 0;
      if (shortBalanceCol !== -1) data[i][shortBalanceCol] = stockMarketData.short_balance || 0;
      if (sblBalanceCol !== -1) data[i][sblBalanceCol] = stockMarketData.sbl_balance || 0;
      if (shortSaleTodayCol !== -1) data[i][shortSaleTodayCol] = stockMarketData.short_sale_today || 0;
      
      // 準備要寫入歷史紀錄的資料
      newHistoryRows.push([dataDate, ticker, foreignNetBuy, trustNetBuy]);
    }
  }

  sheet.getDataRange().setValues(data);
  
  // 將歷史資料一次性寫入新工作表-法人歷史紀錄
  if (newHistoryRows.length > 0) {
    let historySheet = ss.getSheetByName('法人歷史紀錄');
    if (!historySheet) {
      historySheet = ss.insertSheet('法人歷史紀錄');
      historySheet.appendRow(['日期', '股票代碼', '外資買超張數', '投信買超張數']);
    }
    historySheet.getRange(historySheet.getLastRow() + 1, 1, newHistoryRows.length, 4).setValues(newHistoryRows);
    Logger.log(`✅ 已新增 ${newHistoryRows.length} 筆資料至「法人歷史紀錄」。`);
  }

  Logger.log(`✅ 市場數據更新完成！資料日期為：${marketDataMap.data_date}`);
}

//【籌碼面輔助函式】
function fetchLatestMarketData() {
  let latestDateStr = '';
  let institutionalData = [], marginData = [], sblData = [];

  const tryDate = new Date();
  for (let i = 0; i < 5; i++) {
    const dateStr = Utilities.formatDate(tryDate, "Asia/Taipei", "yyyyMMdd");
    const url = `https://www.twse.com.tw/fund/T86?response=json&date=${dateStr}&selectType=ALL`;
    try {
      const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
      const json = JSON.parse(res.getContentText());
      if (json && json.stat === "OK" && json.data && json.data.length > 0) {
        latestDateStr = dateStr;
        institutionalData = json.data;
        Logger.log(`✅ 成功在 ${latestDateStr} 找到法人買賣超資料。`);
        break;
      }
    } catch (e) { /* ... */ }
    tryDate.setDate(tryDate.getDate() - 1);
  }

  if (!latestDateStr) { return null; }
  
  const marginUrl = `https://www.twse.com.tw/exchangeReport/MI_MARGN?response=json&date=${latestDateStr}&selectType=ALL`;
  try {
    const res = UrlFetchApp.fetch(marginUrl, { 'muteHttpExceptions': true });
    const json = JSON.parse(res.getContentText());
    if (json && json.stat === "OK" && json.data && json.data.length > 0) { marginData = json.data; }
  } catch(e) { /* ... */ }

  const sblUrl = `https://www.twse.com.tw/exchangeReport/SBL_LST?response=json&date=${latestDateStr}`;
  try {
      const res = UrlFetchApp.fetch(sblUrl, {'muteHttpExceptions': true});
      const json = JSON.parse(res.getContentText());
      if(json && json.stat === "OK" && json.data && json.data.length > 0) { sblData = json.data; }
  } catch(e) { /* ... */ }

  const marketDataMap = {};
  
  for (const item of institutionalData) {
    const ticker = item[0].trim();
    marketDataMap[ticker] = {
      foreign_net_buy: Math.round(Number(String(item[4]).replace(/,/g, '')) / 1000),
      trust_net_buy: Math.round(Number(String(item[10]).replace(/,/g, '')) / 1000)
    };
  }
  
  for (const item of marginData) {
      const ticker = item[0].trim();
      const margin_balance = Number(String(item[6]).replace(/,/g, ''));
      const short_balance = Number(String(item[12]).replace(/,/g, ''));
      const short_sale_today = Number(String(item[9]).replace(/,/g, ''));
      if (marketDataMap[ticker]) {
        Object.assign(marketDataMap[ticker], { margin_balance, short_balance, short_sale_today });
      } else {
        marketDataMap[ticker] = { margin_balance, short_balance, short_sale_today };
      }
  }

  for (const item of sblData) {
      const ticker = item[0].trim();
      const sbl_balance = Number(String(item[4]).replace(/,/g, ''));
       if (marketDataMap[ticker]) {
        marketDataMap[ticker].sbl_balance = sbl_balance;
      } else {
        marketDataMap[ticker] = { sbl_balance };
      }
  }

  marketDataMap.data_date = latestDateStr;
  return marketDataMap;
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
  // 從第二行開始，忽略標題
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

//【大戶集中度模組】
function updateShareholderConcentration_Debug() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('風控報表');
  if (!sheet) {
    Logger.log('❌ 找不到名為 "風控報表" 的工作表');
    return;
  }
  
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  
  const tickerCol = headers.indexOf('股票代碼');
  const concentrationCol = headers.indexOf('大戶集中度');
  
  if (tickerCol === -1 || concentrationCol === -1) {
    Logger.log('❌ 執行中止：找不到「股票代碼」或「大戶集中度」欄位。');
    return;
  }

  for (let i = 1; i < data.length; i++) {
    const ticker = data[i][tickerCol];
    const isETF = data[i][headers.indexOf('是否為ETF')] === '是';
    if (!ticker || isETF) continue;
    
    // 呼叫使用新 API 的輔助函式
    const shareholdingData = fetchLatestShareholdingData_v2(ticker);
    
    if (shareholdingData) {
      Logger.log(`[股權分散偵錯] ${ticker}: 新 API 回傳的完整持股級距資料: ${JSON.stringify(shareholdingData)}`);

      const majorShareholderLevels = [
        '400,001-600,000',
        '600,001-800,000',
        '800,001-1,000,000',
        '1,000,001以上'
      ];
      
      const concentration = shareholdingData
        .filter(record => majorShareholderLevels.includes(record.HoldingSharesLevel))
        .reduce((sum, record) => sum + record.percent, 0);
        
      data[i][concentrationCol] = concentration.toFixed(2) + '%';
      
    } else {
      data[i][concentrationCol] = '無資料';
    }
  }

  sheet.getDataRange().setValues(data);
  Logger.log('✅ 所有股票的大戶集中度更新完成！');
}

//【大戶集中度輔助函式】
function fetchLatestShareholdingData_v2(ticker) {
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - 10);
  const startDateStr = Utilities.formatDate(startDate, "Asia/Taipei", "yyyy-MM-dd");

  // ★★★ 修正點：更換為新的 API 資料集 ★★★
  const url = `https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockHoldingSharesPer&data_id=${ticker}&start_date=${startDateStr}&token=${FINMIND_API_TOKEN}`;
  
  try {
    const res = UrlFetchApp.fetch(url, { 'muteHttpExceptions': true });
    const json = JSON.parse(res.getContentText());
    if (!json.data || json.data.length === 0) return null;

    const latestDate = json.data.reduce((max, p) => (p.date > max ? p.date : max), json.data[0].date);
    return json.data.filter(record => record.date === latestDate);
  } catch (e) {
    Logger.log(`⚠️ ${ticker}: 抓取股權分散資料時發生錯誤: ${e}`);
    return null;
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

// =================== 測試用函式 (請貼到主程式碼最下方) ===================

/**
 * 這是一個最簡單的推送測試函式，用來驗證主程式的屬性設定和 API 呼叫功能。
 */
function simplePushTest() {
  try {
    Logger.log("--> 開始執行核心功能通暢測試...");

    // 1. 從指令碼屬性安全地讀取金鑰
    const properties = PropertiesService.getScriptProperties();
    const lineChannelToken = properties.getProperty('LINE_CHANNEL_TOKEN');
    const lineUserId = properties.getProperty('LINE_USER_ID');

    // 2. 關鍵檢查：確保金鑰都已設定
    if (!lineChannelToken || !lineUserId) {
      throw new Error("指令碼屬性中的 LINE_CHANNEL_TOKEN 或 LINE_USER_ID 未設定！");
    }
    Logger.log("成功從屬性中讀取 LINE Token 和 User ID。");

    // 3. 準備一則簡單的測試訊息
    const testMessage = "✅ 主程式發送測試成功！\n\n如果看到此訊息，代表你的指令碼屬性設定正確，且 UrlFetchApp 運作正常。";

    // 4. 將訊息推播到 LINE
    pushToLINEforGAS(testMessage, lineChannelToken, lineUserId);

    Logger.log("--> 核心功能通暢測試執行完畢。請查看 LINE 及下方日誌。");

  } catch (error) {
    Logger.log("執行 simplePushTest 時發生錯誤: " + error.message);
    // (可選) 也可以在這裡發送錯誤通知
  }
}


/**
 * 將訊息推播至 LINE (GAS 版本，包含深度日誌)
 */
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
// ========================================================================



























