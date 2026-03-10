#!/usr/bin/env node

console.clear();

const fs = require('node:fs');
const path = require('node:path');

const CONFIG = {
  varUserAccessToken: "",
  varStartDate: "202-03-01",
  varEndDate: "2026-03-09",
  varGranularity: "DAILY",
  varGraphVersion: "v23.0",
  varBusinessManagers: ["1801828990029185", "385352783664075"],
};

function logLine(message) {
  console.log(`\n${message}`);
}

function toUnixTimestamp(dateString, endOfDay = false) {
  const normalized = endOfDay
    ? `${dateString}T23:59:59-03:00`
    : `${dateString}T00:00:00-03:00`;
  const timestamp = Math.floor(new Date(normalized).getTime() / 1000);

  if (Number.isNaN(timestamp)) {
    throw new Error(`Data invalida informada: ${dateString}`);
  }

  return timestamp;
}

function buildHeaders(varUserAccessToken) {
  return {
    Authorization: `Bearer ${varUserAccessToken}`,
  };
}

function normalizePhoneNumber(rawPhone) {
  const onlyDigits = String(rawPhone ?? "").replace(/\D/g, "");
  return onlyDigits || "Sem métricas";
}

async function fetchJson(url, varUserAccessToken) {
  const response = await fetch(url, {
    method: 'GET',
    headers: buildHeaders(varUserAccessToken),
  });

  const payload = await response.json();

  if (!response.ok) {
    const errorMessage = payload?.error?.message || `Falha na requisicao: ${url}`;
    throw new Error(errorMessage);
  }

  return payload;
}

async function fetchAllPages(url, varUserAccessToken) {
  const allItems = [];
  let nextUrl = url;

  while (nextUrl) {
    const payload = await fetchJson(nextUrl, varUserAccessToken);

    if (Array.isArray(payload?.data)) {
      allItems.push(...payload.data);
    }

    nextUrl = payload?.paging?.next || null;
  }

  return allItems;
}

function buildOwnedWabasUrl(varBusinessManagerID) {
  return `https://graph.facebook.com/${CONFIG.varGraphVersion}/${varBusinessManagerID}/owned_whatsapp_business_accounts?limit=500`;
}

function buildPhoneNumbersUrl(varWabaID) {
  return `https://graph.facebook.com/${CONFIG.varGraphVersion}/${varWabaID}/phone_numbers?limit=500`;
}

function buildMetricsUrl(varWabaID, varStartDateUnix, varEndDateUnix) {
  const fields = [
    `pricing_analytics.start(${varStartDateUnix})`,
    `.end(${varEndDateUnix})`,
    `.granularity(${CONFIG.varGranularity})`,
    `.metric_types(VOLUME)`,
    `.dimensions(COUNTRY,PHONE,PRICING_CATEGORY,PRICING_TYPE,TIER)`,
  ].join('');

  return `https://graph.facebook.com/${CONFIG.varGraphVersion}/${varWabaID}?fields=${encodeURIComponent(fields)}`;
}

async function loadWabas(varUserAccessToken) {
  const varWabaList = [];
  const totalsByBusinessManager = {};

  for (const varBusinessManagerID of CONFIG.varBusinessManagers) {
    const items = await fetchAllPages(buildOwnedWabasUrl(varBusinessManagerID), varUserAccessToken);

    const mappedItems = items.map((item) => ({
      varBusinessManagerID,
      wabaID: item?.id ?? '',
      wabaName: item?.name ?? '',
    }));

    totalsByBusinessManager[varBusinessManagerID] = mappedItems.length;
    varWabaList.push(...mappedItems);
  }

  for (const [varBusinessManagerID, total] of Object.entries(totalsByBusinessManager)) {
    logLine(`Total de WABAs para o Business Manager ${varBusinessManagerID}: ${total}`);
  }

  logLine(`Total do array varWabaList: ${varWabaList.length}`);
  return varWabaList;
}

async function loadPhonesForWaba(waba, varUserAccessToken) {
  try {
    const phones = await fetchAllPages(buildPhoneNumbersUrl(waba.wabaID), varUserAccessToken);

    if (!phones.length) {
      return [{
        varBusinessManagerID: waba.varBusinessManagerID,
        wabaID: waba.wabaID,
        wabaName: waba.wabaName,
        Telefone: 'Sem métricas',
      }];
    }

    return phones.map((phone) => ({
      varBusinessManagerID: waba.varBusinessManagerID,
      wabaID: waba.wabaID,
      wabaName: waba.wabaName,
      Telefone: normalizePhoneNumber(phone?.display_phone_number || phone?.verified_name || phone?.id),
    }));
  } catch (error) {
    logLine(`Erro ao consultar telefones da WABA ${waba.wabaID}: ${error.message}`);

    return [{
      varBusinessManagerID: waba.varBusinessManagerID,
      wabaID: waba.wabaID,
      wabaName: waba.wabaName,
      Telefone: 'Sem métricas',
    }];
  }
}

async function loadPhones(varWabaList, varUserAccessToken) {
  const phoneGroups = await Promise.all(
    varWabaList.map((waba) => loadPhonesForWaba(waba, varUserAccessToken))
  );

  const varWabaListPhones = phoneGroups.flat();
  logLine(`Total do array varWabaListPhones: ${varWabaListPhones.length}`);
  return varWabaListPhones;
}

function buildPhonesByWaba(varWabaListPhones) {
  const phonesByWaba = new Map();

  for (const item of varWabaListPhones) {
    if (!phonesByWaba.has(item.wabaID)) {
      phonesByWaba.set(item.wabaID, new Set());
    }

    phonesByWaba.get(item.wabaID).add(item.Telefone);
  }

  return phonesByWaba;
}

function extractMetricDataPoints(payload) {
  const metricContainers = payload?.pricing_analytics?.data;

  if (!Array.isArray(metricContainers)) {
    return [];
  }

  return metricContainers.flatMap((item) =>
    Array.isArray(item?.data_points) ? item.data_points : []
  );
}

function groupReportRows(rows) {
  const groupedRows = new Map();

  for (const row of rows) {
    const key = [
      row.varBusinessManagerID,
      row.wabaID,
      row.wabaName,
      row.Telefone,
      row.Range,
      row.Categoria,
    ].join('||');

    if (!groupedRows.has(key)) {
      groupedRows.set(key, { ...row });
      continue;
    }

    groupedRows.get(key).Quantidade += row.Quantidade;
  }

  return Array.from(groupedRows.values());
}

function buildZeroMetricRows(waba, phonesByWaba, varStartDateUnix, varRange) {
  const phones = phonesByWaba.has(waba.wabaID)
    ? Array.from(phonesByWaba.get(waba.wabaID))
    : ['Sem métricas'];

  return phones.map((Telefone) => ({
    varBusinessManagerID: waba.varBusinessManagerID,
    wabaID: waba.wabaID,
    wabaName: waba.wabaName,
    Telefone,
    Range: varRange,
    Categoria: 'Sem métricas',
    Quantidade: 0,
  }));
}

async function loadMetricsForWaba(waba, phonesByWaba, varUserAccessToken, varStartDateUnix, varEndDateUnix, varRange) {
  try {
    const payload = await fetchJson(
      buildMetricsUrl(waba.wabaID, varStartDateUnix, varEndDateUnix),
      varUserAccessToken
    );

    const dataPoints = extractMetricDataPoints(payload);

    if (!dataPoints.length) {
      return buildZeroMetricRows(waba, phonesByWaba, varStartDateUnix, varRange);
    }

    const rows = dataPoints.map((dataPoint) => ({
      varBusinessManagerID: waba.varBusinessManagerID,
      wabaID: waba.wabaID,
      wabaName: waba.wabaName,
      Telefone: normalizePhoneNumber(dataPoint?.phone_number),
      Range: varRange,
      Categoria: String(dataPoint?.pricing_category ?? 'Sem métricas'),
      Quantidade: Number(dataPoint?.volume) || 0,
    }));

    return groupReportRows(rows);
  } catch (error) {
    logLine(`Erro ao consultar metricas da WABA ${waba.wabaID}: ${error.message}`);
    return buildZeroMetricRows(waba, phonesByWaba, varStartDateUnix);
  }
}

async function loadMetrics(varWabaList, varWabaListPhones, varUserAccessToken, varStartDateUnix, varEndDateUnix, varRange) {
  const phonesByWaba = buildPhonesByWaba(varWabaListPhones);
  const metricGroups = await Promise.all(
    varWabaList.map((waba) =>
      loadMetricsForWaba(
        waba,
        phonesByWaba,
        varUserAccessToken,
        varStartDateUnix,
        varEndDateUnix,
		varRange
      )
    )
  );

  const varReportList = groupReportRows(metricGroups.flat());
  logLine(`Total de registros no varReportList: ${varReportList.length}`);
  return varReportList;
}

function escapeCsvValue(value) {
  const stringValue = String(value ?? '').replace(/;/g, ',');
  return stringValue;
}

function buildCsvContent(varReportList) {
  const lines = [
    'BM_ID;wabaID;wabaName;Telefone;Range;Categoria;Quantidade',
  ];

  for (const row of varReportList) {
    lines.push([
      escapeCsvValue(row.varBusinessManagerID),
      escapeCsvValue(row.wabaID),
      escapeCsvValue(row.wabaName),
      escapeCsvValue(row.Telefone),
      escapeCsvValue(row.Range),
      escapeCsvValue(row.Categoria),
      escapeCsvValue(row.Quantidade),
    ].join(';'));
  }

  return lines.join('\n');
}

function getTodayFilePrefix() {
  const now = new Date();
  const year = now.getFullYear();
  const month = String(now.getMonth() + 1).padStart(2, '0');
  const day = String(now.getDate()).padStart(2, '0');
  return `${year}-${month}-${day}`;
}

function getUniqueOutputFilePath() {
  const baseName = `${getTodayFilePrefix()}-MetaMetricsOutput.csv`;
  const initialPath = path.join(process.cwd(), baseName);

  if (!fs.existsSync(initialPath)) {
    return initialPath;
  }

  let suffix = 1;

  while (true) {
    const candidateName = `${getTodayFilePrefix()}-MetaMetricsOutput-${String(suffix).padStart(2, '0')}.csv`;
    const candidatePath = path.join(process.cwd(), candidateName);

    if (!fs.existsSync(candidatePath)) {
      return candidatePath;
    }

    suffix += 1;
  }
}

function saveReportCsv(varReportList) {
  const outputFilePath = getUniqueOutputFilePath();
  const csvContent = buildCsvContent(varReportList);

  fs.writeFileSync(outputFilePath, csvContent, 'utf8');
  logLine(`Arquivo CSV gerado: ${outputFilePath}`);
  return outputFilePath;
}

function printReportTable(varReportList) {
  logLine('Conteudo do varReportList:');
  console.log("BM_ID;wabaID;wabaName;Telefone;Range;Categoria;Quantidade");

  for (const row of varReportList) {
    console.log([
      escapeCsvValue(row.varBusinessManagerID),
      escapeCsvValue(row.wabaID),
      escapeCsvValue(row.wabaName),
      escapeCsvValue(row.Telefone),
      escapeCsvValue(row.Range),
      escapeCsvValue(row.Categoria),
      escapeCsvValue(row.Quantidade),
    ].join(';'));
  }
}

async function main() {
  if (!CONFIG.varUserAccessToken) {
    throw new Error('Preencha CONFIG.varUserAccessToken com um token valido antes de executar.');
  }

  const varStartDateUnix = toUnixTimestamp(CONFIG.varStartDate);
  const varEndDateUnix = toUnixTimestamp(CONFIG.varEndDate, true);

  const varRange = CONFIG.varStartDate + " a " + CONFIG.varEndDate;

  logLine(`varStartDateUnix: ${varStartDateUnix}`);
  logLine(`varEndDateUnix: ${varEndDateUnix}`);

  const varWabaList = await loadWabas(CONFIG.varUserAccessToken);
  const varWabaListPhones = await loadPhones(varWabaList, CONFIG.varUserAccessToken);
  const varReportList = await loadMetrics(
    varWabaList,
    varWabaListPhones,
    CONFIG.varUserAccessToken,
    varStartDateUnix,
    varEndDateUnix,
	varRange
  );

  printReportTable(varReportList);
  saveReportCsv(varReportList);
}

main().catch((error) => {
  logLine(`Erro: ${error.message}`);
  process.exitCode = 1;
});
