import sys
import os
import json
import traceback
import pyodbc
import warnings
from datetime import datetime
from typing import List, Dict, Tuple, Optional

import pandas as pd
import numpy as np

try:
    from sklearn.ensemble import RandomForestRegressor
    from sklearn.preprocessing import LabelEncoder
    SKLEARN_AVAILABLE = True
except ImportError:
    SKLEARN_AVAILABLE = False

from PySide6.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QFileDialog, QTableView, QHeaderView,
    QStackedWidget, QComboBox, QGroupBox, QMessageBox, QFrame,
    QDialog, QFormLayout, QGridLayout, QScrollArea, QSizePolicy,
    QLineEdit, QCheckBox, QToolBar, QMenu
)
from PySide6.QtCore import Qt, QAbstractTableModel, QModelIndex, QSortFilterProxyModel, QThread, Signal, QTimer
from PySide6.QtGui import QFont, QAction, QPainter, QColor

from PySide6.QtCharts import (
    QChart, QChartView, QBarSet, QBarSeries, QBarCategoryAxis, 
    QPieSeries, QHorizontalBarSeries, QValueAxis
)

# ==========================================
# 0. SQL CONFIGURATION
# ==========================================
SQL_QUERY = """
WITH
params AS (
    SELECT
        DATE '2025-10-01' AS start_date,
        DATE '2027-10-01' AS end_date
    FROM dual
),

filtered_setl_runs AS (
    SELECT /*+ MATERIALIZE */
        gar.garunidx,
        gar.rundate,
        gar.descr,
        gar.setltype
    FROM ga_setl_run gar
    CROSS JOIN params p
    WHERE gar.rundate >= p.start_date
      AND gar.rundate <  p.end_date
      AND gar.setltype IN ('1','2')
),

/* -------------------------------------------------------------------------
   Receipt rows moved up to build a fast valid_lots filter early.
------------------------------------------------------------------------- */
receipt_rows AS (
    SELECT
        recv.galotidx,
        recv.gablockidx,
        recv.ictrxhdridx,
        recv.tagid,
        recv.icqnt,
        recv.receivedate,
        recv.commodity,
        recv.variety,
        recv.style,
        recv.sizename,
        recv.color,
        recv.grade,
        UPPER(TRIM(recv.commodity)) AS commodity_norm,
        UPPER(TRIM(recv.variety)) AS variety_norm,
        UPPER(TRIM(recv.style)) AS style_norm,
        UPPER(TRIM(recv.sizename)) AS sizename_norm,
        UPPER(TRIM(recv.color)) AS color_norm,
        UPPER(TRIM(recv.grade)) AS grade_norm
    FROM "COMPANY_1"."IC_TRX_PRODUCT_VIEW" recv
    CROSS JOIN params p
    WHERE recv.ictrxtype = '1'
      AND recv.receivedate >= p.start_date
      AND recv.receivedate <  p.end_date
),

base_receipts AS (
    SELECT /*+ MATERIALIZE */
        'RECEIPT' AS record_type,
        gl.galotidx AS lot_index,
        gl.id AS lot_id,
        gl.descr AS lot_descr,
        g_name.lastconame AS grower_name,
        thdr.ref AS reference,
        gl.closedate AS lot_close_date,
        gb.id AS block_name,
        recv.tagid AS pallet_tag_id,
        recv.commodity_norm AS commodity,
        recv.variety_norm AS variety,
        recv.style_norm AS style,
        recv.sizename_norm AS sizename,
        recv.color_norm AS color,
        recv.grade_norm AS grade,
        recv.receivedate AS received_date,
        SUM(recv.icqnt) AS qty_received,
        SUM(recv.icqnt) / NULLIF(
            SUM(SUM(recv.icqnt)) OVER (
                PARTITION BY gl.galotidx
            ),
            0
        ) AS pallet_lot_ratio,
        SUM(recv.icqnt) / NULLIF(
            SUM(SUM(recv.icqnt)) OVER (
                PARTITION BY
                    gl.galotidx,
                    recv.commodity_norm,
                    recv.variety_norm,
                    recv.style_norm,
                    recv.sizename_norm,
                    recv.color_norm,
                    recv.grade_norm
            ),
            0
        ) AS pallet_full_attr_ratio,
        SUM(recv.icqnt) / NULLIF(
            SUM(SUM(recv.icqnt)) OVER (
                PARTITION BY
                    gl.galotidx,
                    recv.commodity_norm,
                    recv.variety_norm
            ),
            0
        ) AS pallet_variety_ratio,
        SUM(recv.icqnt) / NULLIF(
            SUM(SUM(recv.icqnt)) OVER (
                PARTITION BY gl.galotidx
            ),
            0
        ) AS pallet_lot_fallback_ratio
    FROM receipt_rows recv
    INNER JOIN ga_lot gl
      ON recv.galotidx = gl.galotidx
    LEFT JOIN ga_block gb
      ON recv.gablockidx = gb.gablockidx
    LEFT JOIN fc_name g_name
      ON gb.growernameidx = g_name.nameidx
    LEFT JOIN "COMPANY_1"."IC_TRX_HEADER" thdr
      ON recv.ictrxhdridx = thdr.ictrxhdridx
    GROUP BY
        gl.galotidx,
        gl.id,
        gl.descr,
        g_name.lastconame,
        thdr.ref,
        gl.closedate,
        gb.id,
        recv.tagid,
        recv.commodity_norm,
        recv.variety_norm,
        recv.style_norm,
        recv.sizename_norm,
        recv.color_norm,
        recv.grade_norm,
        recv.receivedate
),

valid_lots AS (
    SELECT DISTINCT lot_index 
    FROM base_receipts
),

/* -------------------------------------------------------------------------
   RawReportData kept 100% original with INNER JOINs to guarantee 
   that Sales, Advances, and Tariff Metrics process flawlessly.
------------------------------------------------------------------------- */
RawReportData AS (
    /* 1. Receiving Charges */
    SELECT /*+ MATERIALIZE */
        gl.galotidx,
        gl.id AS lotid,
        gl.descr AS lotdescr,
        gl.closedate AS lotclosedate,
        gl.histflag AS lothistflag,
        icp.productidx,
        fcc.fcchargeidx,
        fcc.id AS chargeid,
        fcc.descr AS chargedescr,
        fcc.ratetype AS chargeratetype,
        fcc.orderby AS chargeorderby,
        '2' AS trxtype,
        DECODE(ich.ictrxtype, '3', 4, 2) AS sourceidx,
        gb.gablockidx,
        gb.id AS growerblockid,
        gb.name AS blockname,
        fn.id AS growerid,
        fn.lastconame AS growername,
        DECODE(ich.ictrxtype, '3', ich.trxdate, TO_DATE(NULL)) AS refdate,
        DECODE(ich.ictrxtype, '3', TO_CHAR(ich.icrunidx), NULL) AS ref,
        gb.name AS descr,
        icc.garunidx,
        0 AS invcicqnt,
        0 AS invcqnt,
        0 AS saleamt,
        SUM(icd.foreignamt) AS chgamt,
        SUM(icc.qnt) AS chgqnt,
        NULL AS uom
    FROM ic_trx_charge icc
    JOIN ic_trx_detail icd
      ON icc.ictrxhdridx = icd.ictrxhdridx
     AND icc.ictrxdtlseq = icd.ictrxdtlseq
    JOIN ic_trx_header ich
      ON icd.ictrxhdridx = ich.ictrxhdridx
    JOIN ga_block gb
      ON icd.gablockidx = gb.gablockidx
    JOIN fc_name fn
      ON gb.growernameidx = fn.nameidx
    JOIN filtered_setl_runs gar
      ON icc.garunidx = gar.garunidx
    JOIN fc_charge_code fcc
      ON icc.fcchargeidx = fcc.fcchargeidx
    LEFT JOIN ga_lot gl
      ON icd.galotidx = gl.galotidx
    LEFT JOIN ic_trx_product icp
      ON icc.ictrxhdridx = icp.ictrxhdridx
     AND icc.gaproductautochgseq = icp.ictrxdtlseq
    WHERE icd.trxtype = '2'
      AND fcc.chargetype = 3
    GROUP BY
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        icp.productidx,
        fcc.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        DECODE(ich.ictrxtype, '3', 4, 2),
        gb.gablockidx,
        gb.id,
        gb.name,
        fn.id,
        fn.lastconame,
        DECODE(ich.ictrxtype, '3', ich.trxdate, TO_DATE(NULL)),
        DECODE(ich.ictrxtype, '3', TO_CHAR(ich.icrunidx), NULL),
        gb.name,
        icc.garunidx

    UNION ALL

    /* 2. AR Sales Receipts / Liquidations */
    SELECT
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        arp.productidx,
        TO_NUMBER(NULL),
        NULL,
        NULL,
        NULL,
        0,
        '1',
        1,
        gb.gablockidx,
        gb.id,
        gb.name,
        gwr_name.id,
        gwr_name.lastconame,
        arh.shipdatetime,
        arh.sono,
        cust_name.lastconame,
        ard.garunidx,
        SUM(arp.icqnt),
        SUM(arp.qnt),
        SUM(arp.gasetlamt),
        0,
        0,
        uom.uom
    FROM ar_trx_detail ard
    JOIN ar_trx_product arp
      ON ard.artrxhdridx = arp.artrxhdridx
     AND ard.artrxdtlseq = arp.artrxdtlseq
    JOIN ar_trx_header arh
      ON ard.artrxhdridx = arh.artrxhdridx
    JOIN ar_trx_line arl
      ON ard.artrxhdridx = arl.artrxhdridx
     AND ard.artrxlinetrxtype = arl.artrxlinetrxtype
     AND ard.artrxlineseq = arl.artrxlineseq
    JOIN fc_name cust_name
      ON arh.custnameidx = cust_name.nameidx
    JOIN ga_block gb
      ON arp.gablockidx = gb.gablockidx
    JOIN fc_unit_of_measure uom
      ON arl.uomidx = uom.uomidx
    JOIN fc_name gwr_name
      ON gb.growernameidx = gwr_name.nameidx
    JOIN filtered_setl_runs gar
      ON ard.garunidx = gar.garunidx
    LEFT JOIN ic_inventory ici
      ON arp.inventoryidx = ici.inventoryidx
    LEFT JOIN ga_lot gl
      ON ici.galotidx = gl.galotidx
    WHERE arp.filltype IN (2,4,7)
    GROUP BY
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        arp.productidx,
        gb.gablockidx,
        gb.id,
        gb.name,
        gwr_name.id,
        gwr_name.lastconame,
        arh.shipdatetime,
        arh.sono,
        cust_name.lastconame,
        ard.garunidx,
        uom.uom

    UNION ALL

    /* 3. AR Sales Associated Charges */
    SELECT
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        arp.productidx,
        arc.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        '2',
        1,
        gb.gablockidx,
        gb.id,
        gb.name,
        gwr_name.id,
        gwr_name.lastconame,
        arh.shipdatetime,
        arh.sono,
        cust_name.lastconame,
        ard.garunidx,
        0,
        0,
        0,
        SUM(ard.foreignamt),
        SUM(arc.qnt),
        NULL
    FROM ar_trx_header arh
    JOIN ar_trx_line arl
      ON arh.artrxhdridx = arl.artrxhdridx
    JOIN ar_trx_detail ard
      ON arl.artrxhdridx = ard.artrxhdridx
     AND arl.artrxlinetrxtype = ard.artrxlinetrxtype
     AND arl.artrxlineseq = ard.artrxlineseq
    JOIN ar_trx_charge arc
      ON ard.artrxhdridx = arc.artrxhdridx
     AND ard.artrxdtlseq = arc.artrxdtlseq
    JOIN fc_name cust_name
      ON arh.custnameidx = cust_name.nameidx
    JOIN fc_charge_code fcc
      ON arc.fcchargeidx = fcc.fcchargeidx
    JOIN ga_block gb
      ON arc.gablockidx = gb.gablockidx
    JOIN fc_name gwr_name
      ON gb.growernameidx = gwr_name.nameidx
    JOIN filtered_setl_runs gar
      ON ard.garunidx = gar.garunidx
    LEFT JOIN ar_trx_product arp
      ON arc.artrxhdridx = arp.artrxhdridx
     AND arc.gaproductautochgseq = arp.artrxdtlseq
    LEFT JOIN ga_lot gl
      ON arc.galotidx = gl.galotidx
    WHERE arl.artrxlinetrxtype IN ('1','2')
      AND ard.trxtype = '2'
      AND fcc.chargetype = 3
    GROUP BY
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        arp.productidx,
        arc.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        gb.gablockidx,
        gb.id,
        gb.name,
        gwr_name.id,
        gwr_name.lastconame,
        arh.shipdatetime,
        arh.sono,
        cust_name.lastconame,
        ard.garunidx

    UNION ALL

    /* 4. Automated Settlement Charges */
    SELECT
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        DECODE(gac.sourceidx, 10, arp.productidx, TO_NUMBER(NULL)),
        gac.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        '2',
        DECODE(gac.sourceidx, 10, 1, 3),
        gb.gablockidx,
        gb.id,
        gb.name,
        fn.id,
        fn.lastconame,
        DECODE(gac.sourceidx, 10, arh.shipdatetime, gar.rundate),
        DECODE(gac.sourceidx, 10, arh.sono, TO_CHAR(gac.garunidx)),
        DECODE(gac.sourceidx, 10, cust.lastconame, gar.descr),
        gac.garunidx,
        0,
        0,
        0,
        SUM(gac.amt),
        SUM(gac.qnt),
        NULL
    FROM ga_trx_auto_setl_charge gac
    JOIN filtered_setl_runs gar
      ON gac.garunidx = gar.garunidx
    JOIN fc_charge_code fcc
      ON gac.fcchargeidx = fcc.fcchargeidx
    LEFT JOIN ga_block gb
      ON gac.gablockidx = gb.gablockidx
    LEFT JOIN ga_lot gl
      ON gac.galotidx = gl.galotidx
    LEFT JOIN fc_name fn
      ON gb.growernameidx = fn.nameidx
    LEFT JOIN ar_trx_header arh
      ON gac.keyhdr = arh.artrxhdridx
    LEFT JOIN fc_name cust
      ON arh.custnameidx = cust.nameidx
    LEFT JOIN ar_trx_product arp
      ON gac.keyhdr = arp.artrxhdridx
     AND gac.keyseq = arp.artrxdtlseq
    GROUP BY
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        DECODE(gac.sourceidx, 10, arp.productidx, TO_NUMBER(NULL)),
        gac.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        DECODE(gac.sourceidx, 10, 1, 3),
        gb.gablockidx,
        gb.id,
        gb.name,
        fn.id,
        fn.lastconame,
        DECODE(gac.sourceidx, 10, arh.shipdatetime, gar.rundate),
        DECODE(gac.sourceidx, 10, arh.sono, TO_CHAR(gac.garunidx)),
        DECODE(gac.sourceidx, 10, cust.lastconame, gar.descr),
        gac.garunidx

    UNION ALL

    /* 5. Pool/Contract Run Detail Charges */
    SELECT
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        TO_NUMBER(NULL),
        crdv.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        '2',
        3,
        crdv.gablockidx,
        crdv.blockid,
        crdv.blockname,
        crdv.growernameid,
        crdv.growername,
        crdv.refdate,
        crdv.ref,
        SUBSTR(crdv.descr, 1, 40),
        crdv.garunidx,
        0,
        0,
        0,
        SUM(crdv.ctamt),
        SUM(crdv.qnt),
        NULL
    FROM ct_run_detail_view crdv
    JOIN ga_block gb
      ON crdv.gablockidx = gb.gablockidx
    JOIN fc_charge_code fcc
      ON crdv.fcchargeidx = fcc.fcchargeidx
    JOIN filtered_setl_runs gar
      ON crdv.garunidx = gar.garunidx
    LEFT JOIN ga_lot gl
      ON crdv.galotidx = gl.galotidx
    WHERE crdv.ctdesttype = '1'
      AND crdv.growernameidx IS NOT NULL
    GROUP BY
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        crdv.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        crdv.gablockidx,
        crdv.blockid,
        crdv.blockname,
        crdv.growernameid,
        crdv.growername,
        crdv.refdate,
        crdv.ref,
        SUBSTR(crdv.descr, 1, 40),
        crdv.garunidx

    UNION ALL

    /* 6. Inventory Issuance Transactions */
    SELECT
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        ici.productidx,
        TO_NUMBER(NULL),
        NULL,
        NULL,
        NULL,
        0,
        '1',
        4,
        gb.gablockidx,
        gb.id,
        gb.name,
        gwr_name.id,
        gwr_name.lastconame,
        icih.issuedatetime,
        TO_CHAR(icid.icrunidx),
        gb.name,
        icid.garunidx,
        SUM(icip.icqnt),
        SUM(icip.qnt),
        SUM(icip.gasetlamt),
        0,
        0,
        uom.uom
    FROM ic_trx_issue_detail icid
    JOIN ic_trx_issue_product icip
      ON icid.icissuehdridx = icip.icissuehdridx
     AND icid.icissueseq = icip.icissueseq
    JOIN ic_trx_issue_header icih
      ON icid.icissuehdridx = icih.icissuehdridx
    JOIN ic_inventory ici
      ON icip.inventoryidx = ici.inventoryidx
    JOIN ga_block gb
      ON ici.gablockidx = gb.gablockidx
    JOIN fc_unit_of_measure uom
      ON icip.uomidx = uom.uomidx
    JOIN fc_name gwr_name
      ON gb.growernameidx = gwr_name.nameidx
    JOIN filtered_setl_runs gar
      ON icid.garunidx = gar.garunidx
    LEFT JOIN ga_lot gl
      ON ici.galotidx = gl.galotidx
    GROUP BY
        gl.galotidx,
        gl.id,
        gl.descr,
        gl.closedate,
        gl.histflag,
        ici.productidx,
        gb.gablockidx,
        gb.id,
        gb.name,
        gwr_name.id,
        gwr_name.lastconame,
        icih.issuedatetime,
        TO_CHAR(icid.icrunidx),
        gb.name,
        icid.garunidx,
        uom.uom

    UNION ALL

    /* 7. Inventory Issuance Charges */
    SELECT
        gl.galotidx,
        gl.id,
        gl.descr,
        TO_DATE(NULL) AS lotclosedate,
        CAST(NULL AS VARCHAR2(1)) AS lothistflag,
        ici.productidx,
        icic.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        '2',
        4,
        gb.gablockidx,
        gb.id,
        gb.name,
        gwr_name.id,
        gwr_name.lastconame,
        icih.issuedatetime,
        TO_CHAR(icid.icrunidx),
        gb.name,
        icid.garunidx,
        0,
        0,
        0,
        SUM(icic.foreignamt),
        SUM(icic.qnt),
        NULL
    FROM ic_trx_issue_header icih
    JOIN ic_trx_issue_detail icid
      ON icih.icissuehdridx = icid.icissuehdridx
    JOIN ic_trx_issue_charge icic
      ON icid.icissuehdridx = icic.icissuehdridx
     AND icid.icissueseq = icic.icissueseq
    JOIN fc_charge_code fcc
      ON icic.fcchargeidx = fcc.fcchargeidx
    JOIN ga_block gb
      ON icic.gablockidx = gb.gablockidx
    JOIN fc_name gwr_name
      ON gb.growernameidx = gwr_name.nameidx
    JOIN filtered_setl_runs gar
      ON icid.garunidx = gar.garunidx
    LEFT JOIN ga_lot gl
      ON icic.galotidx = gl.galotidx
    LEFT JOIN ic_trx_issue_product icip
      ON icic.icissuehdridx = icip.icissuehdridx
     AND icic.producticissueseq = icip.icissueseq
    LEFT JOIN ic_inventory ici
      ON icip.inventoryidx = ici.inventoryidx
    WHERE icid.trxtype = '2'
      AND fcc.chargetype = 3
    GROUP BY
        gl.galotidx,
        gl.id,
        gl.descr,
        ici.productidx,
        icic.fcchargeidx,
        fcc.id,
        fcc.descr,
        fcc.ratetype,
        fcc.orderby,
        gb.gablockidx,
        gb.id,
        gb.name,
        gwr_name.id,
        gwr_name.lastconame,
        icih.issuedatetime,
        TO_CHAR(icid.icrunidx),
        gb.name,
        icid.garunidx
),

/* -------------------------------------------------------------------------
   TariffRule & LotMetrics kept 100% original.
------------------------------------------------------------------------- */
TariffRule AS (
    SELECT
        garunidx,
        1 AS has_tariff
    FROM RawReportData
    WHERE UPPER(TRIM(chargedescr)) IN ('RECIPROCAL TARRIFFS', 'RECIPROCAL TARIFFS')
    GROUP BY garunidx
),

LotMetrics AS (
    SELECT
        lotid,
        SUM(saleamt) AS lot_total_sale,
        SUM(chgamt) AS lot_total_charges,
        SUM(
            CASE
                WHEN UPPER(TRIM(chargedescr)) LIKE '%ADVANCE%'
                  OR UPPER(TRIM(chargedescr)) LIKE '%PAYMENT ON ACCOUNT%'
                THEN chgamt
                ELSE 0
            END
        ) AS lot_total_advances,
        SUM(invcqnt) AS lot_total_qty
    FROM RawReportData
    GROUP BY lotid
),

sales_product_ids AS (
    SELECT DISTINCT productidx
    FROM RawReportData
    WHERE trxtype = '1'
      AND saleamt <> 0
      AND productidx IS NOT NULL
),

distinct_item_attributes AS (
    SELECT DISTINCT
        p.productidx,
        UPPER(TRIM(p.commodity)) AS commodity,
        UPPER(TRIM(p.variety)) AS variety,
        UPPER(TRIM(p.style)) AS style,
        UPPER(TRIM(p.sizename)) AS sizename,
        UPPER(TRIM(p.color)) AS color,
        UPPER(TRIM(p.grade)) AS grade
    FROM "COMPANY_1"."IC_TRX_PRODUCT_VIEW" p
    JOIN sales_product_ids s
      ON p.productidx = s.productidx
),

receipt_attributes AS (
    SELECT DISTINCT
        lot_index,
        commodity,
        variety,
        style,
        sizename,
        color,
        grade
    FROM base_receipts
),

receipt_varieties AS (
    SELECT DISTINCT
        lot_index,
        commodity,
        variety
    FROM base_receipts
),

sales_grouped AS (
    SELECT /*+ MATERIALIZE */
        r.galotidx AS lot_index,
        a.commodity,
        a.variety,
        a.style,
        a.sizename,
        a.color,
        a.grade,
        MAX(r.garunidx) AS settlement_run_idx,
        SUM(r.saleamt) AS total_sales_amt
    FROM RawReportData r
    JOIN distinct_item_attributes a
      ON r.productidx = a.productidx
    WHERE r.trxtype = '1'
      AND r.saleamt <> 0
    GROUP BY
        r.galotidx,
        a.commodity,
        a.variety,
        a.style,
        a.sizename,
        a.color,
        a.grade
),

sales_classified AS (
    SELECT
        sg.*,
        CASE
            WHEN ra.lot_index IS NOT NULL THEN 1
            WHEN rv.lot_index IS NOT NULL THEN 2
            ELSE 3
        END AS sales_tier
    FROM sales_grouped sg
    LEFT JOIN receipt_attributes ra
      ON sg.lot_index = ra.lot_index
     AND sg.commodity = ra.commodity
     AND sg.variety = ra.variety
     AND sg.style = ra.style
     AND sg.sizename = ra.sizename
     AND (
            sg.color = ra.color
         OR (sg.color IS NULL AND ra.color IS NULL)
     )
     AND (
            sg.grade = ra.grade
         OR (sg.grade IS NULL AND ra.grade IS NULL)
     )
    LEFT JOIN receipt_varieties rv
      ON sg.lot_index = rv.lot_index
     AND sg.commodity = rv.commodity
     AND sg.variety = rv.variety
),

tier1_sales AS (
    SELECT
        lot_index,
        commodity,
        variety,
        style,
        sizename,
        color,
        grade,
        MAX(settlement_run_idx) AS settlement_run_idx,
        SUM(total_sales_amt) AS t1_amt
    FROM sales_classified
    WHERE sales_tier = 1
    GROUP BY
        lot_index,
        commodity,
        variety,
        style,
        sizename,
        color,
        grade
),

tier2_sales AS (
    SELECT
        lot_index,
        commodity,
        variety,
        MAX(settlement_run_idx) AS settlement_run_idx,
        SUM(total_sales_amt) AS t2_amt
    FROM sales_classified
    WHERE sales_tier = 2
    GROUP BY
        lot_index,
        commodity,
        variety
),

tier3_sales AS (
    SELECT
        lot_index,
        MAX(settlement_run_idx) AS settlement_run_idx,
        SUM(total_sales_amt) AS t3_amt
    FROM sales_classified
    WHERE sales_tier = 3
    GROUP BY lot_index
),

lot_sales_total AS (
    SELECT
        r.galotidx AS lot_index,
        MAX(r.garunidx) AS settlement_run_idx,
        SUM(r.saleamt) AS lot_sales_amt
    FROM RawReportData r
    WHERE r.trxtype = '1'
      AND r.saleamt <> 0
    GROUP BY r.galotidx
),

tier_pool_totals_by_lot AS (
    SELECT
        l.lot_index,
        l.lot_sales_amt,
        NVL(t1.t1_sum, 0) AS t1_sum,
        NVL(t2.t2_sum, 0) AS t2_sum,
        NVL(t3.t3_sum, 0) AS t3_sum,
        l.lot_sales_amt
            - (
                NVL(t1.t1_sum, 0)
              + NVL(t2.t2_sum, 0)
              + NVL(t3.t3_sum, 0)
            ) AS t4_remaining_amt
    FROM lot_sales_total l
    LEFT JOIN (
        SELECT lot_index, SUM(t1_amt) AS t1_sum
        FROM tier1_sales
        GROUP BY lot_index
    ) t1
      ON l.lot_index = t1.lot_index
    LEFT JOIN (
        SELECT lot_index, SUM(t2_amt) AS t2_sum
        FROM tier2_sales
        GROUP BY lot_index
    ) t2
      ON l.lot_index = t2.lot_index
    LEFT JOIN (
        SELECT lot_index, SUM(t3_amt) AS t3_sum
        FROM tier3_sales
        GROUP BY lot_index
    ) t3
      ON l.lot_index = t3.lot_index
),

/* -------------------------------------------------------------------------
   MODIFIED: charge_alloc updated to pull in pending charges using a 
   LEFT JOIN and IS NULL check, while keeping performance high via valid_lots.
------------------------------------------------------------------------- */
charge_alloc AS (
    SELECT
        'Inventory' AS charge_source,
        lot.galotidx AS lot_index,
        fc.descr AS charge_name,
        SUM(icd.foreignamt) AS total_pool_amt
    FROM ic_trx_detail icd
    JOIN ic_trx_charge charge
      ON icd.ictrxhdridx = charge.ictrxhdridx
     AND icd.ictrxdtlseq = charge.ictrxdtlseq
    JOIN fc_charge_code fc
      ON charge.fcchargeidx = fc.fcchargeidx
    LEFT JOIN ga_lot lot
      ON icd.galotidx = lot.galotidx
    JOIN valid_lots vl 
      ON lot.galotidx = vl.lot_index
    LEFT JOIN filtered_setl_runs gar
      ON charge.garunidx = gar.garunidx
    WHERE fc.chargetype = 3
      AND (NVL(charge.garunidx, 0) = 0 OR gar.garunidx IS NOT NULL)
    GROUP BY
        lot.galotidx,
        fc.descr

    UNION ALL

    SELECT
        'CT/AP' AS charge_source,
        lot.galotidx AS lot_index,
        ctdv.chargedescr AS charge_name,
        SUM(ctdv.ctamt) AS total_pool_amt
    FROM ct_run_detail_view ctdv
    LEFT JOIN ga_lot lot
      ON ctdv.lotid = lot.id
    JOIN valid_lots vl 
      ON lot.galotidx = vl.lot_index
    LEFT JOIN filtered_setl_runs gar
      ON ctdv.garunidx = gar.garunidx
    WHERE (NVL(ctdv.garunidx, 0) = 0 OR gar.garunidx IS NOT NULL)
    GROUP BY
        lot.galotidx,
        ctdv.chargedescr

    UNION ALL

    SELECT
        'Auto Setl' AS charge_source,
        lot.galotidx AS lot_index,
        fc.descr AS charge_name,
        SUM(gac.amt) AS total_pool_amt
    FROM ga_trx_auto_setl_charge gac
    LEFT JOIN fc_charge_code fc
      ON gac.fcchargeidx = fc.fcchargeidx
    LEFT JOIN ga_lot lot
      ON gac.galotidx = lot.galotidx
    JOIN valid_lots vl 
      ON lot.galotidx = vl.lot_index
    LEFT JOIN filtered_setl_runs gar
      ON gac.garunidx = gar.garunidx
    WHERE NVL(gac.gldeletecode, 'N') = 'N'
      AND (NVL(gac.garunidx, 0) = 0 OR gar.garunidx IS NOT NULL)
    GROUP BY
        lot.galotidx,
        fc.descr

    UNION ALL

    SELECT
        'Commission' AS charge_source,
        lot.galotidx AS lot_index,
        fc.descr AS charge_name,
        SUM(ard.foreignamt) AS total_pool_amt
    FROM ar_trx_charge charge
    JOIN ar_trx_detail ard
      ON charge.artrxhdridx = ard.artrxhdridx
     AND charge.artrxdtlseq = ard.artrxdtlseq
    JOIN fc_charge_code fc
      ON charge.fcchargeidx = fc.fcchargeidx
    LEFT JOIN ga_lot lot
      ON charge.galotidx = lot.galotidx
    JOIN valid_lots vl 
      ON lot.galotidx = vl.lot_index
    LEFT JOIN filtered_setl_runs gar
      ON ard.garunidx = gar.garunidx
    WHERE fc.chargetype = 3
      AND (NVL(ard.garunidx, 0) = 0 OR gar.garunidx IS NOT NULL)
    GROUP BY
        lot.galotidx,
        fc.descr
),

charge_totals AS (
    SELECT
        lot_index,
        SUM(total_pool_amt) AS total_lot_charges_pool
    FROM charge_alloc
    GROUP BY lot_index
),

pivoted_receipts AS (
    SELECT *
    FROM (
        SELECT
            b.record_type,
            b.lot_index,
            b.lot_id,
            b.lot_descr,
            b.grower_name,
            b.reference,
            b.lot_close_date,
            b.received_date,
            b.block_name,
            b.pallet_tag_id,
            b.commodity,
            b.variety,
            b.style,
            b.sizename,
            b.color,
            b.grade,
            b.qty_received,
            b.pallet_lot_ratio,
            b.pallet_full_attr_ratio,
            b.pallet_variety_ratio,
            b.pallet_lot_fallback_ratio,
            ct.total_lot_charges_pool,
            UPPER(TRIM(alloc.charge_name)) AS matrix_charge_name,
            ROUND(b.pallet_lot_ratio * alloc.total_pool_amt, 2) AS allocated_charge_amount
        FROM base_receipts b
        JOIN charge_alloc alloc
          ON b.lot_index = alloc.lot_index
        JOIN charge_totals ct
          ON b.lot_index = ct.lot_index
    )
    PIVOT (
        SUM(allocated_charge_amount)
        FOR matrix_charge_name IN (
            'COLD STORAGE'                  AS cold_storage,
            'CUSTOMS'                       AS customs,
            'CUSTOMS - EXAMS'               AS customs_exams,
            'DEMURRAGE'                     AS demurrage,
            'FREIGHT CHARGES'               AS freight_charges,
            'GROWER ADVANCE'                AS grower_advance,
            'INLAND FREIGHT'                AS inland_freight,
            'INSPECTION CHARGES'            AS inspection_charges,
            'OCEAN FREIGHT'                 AS ocean_freight,
            'QC ASSESSMENT CHARGE'          AS qc_assessment,
            'SALES MARGIN'                  AS sales_margin,
            'TEMPERATURE RECORDER'          AS temp_recorder,
            'GROWER PAYMENT ON ACCOUNT'     AS grower_pmt_on_acct,
            'FUMIGATION'                    AS fumigation,
            'COLD TREATMENT'                AS cold_treatment,
            'GROWER ALLOWANCE & ADJUSTMENT' AS grower_allow_adjust,
            'DUMP/REPACK FEES'              AS dump_repack_fees,
            'SEAL'                          AS seal,
            'FREIGHT & HANDLING CHARGES'    AS freight_handling,
            'CUSTOMER REBATE'               AS customer_rebate,
            'CUSTOMER ALLOWANCE'            AS customer_allowance,
            'SALES MARGIN - FLAT RATE'      AS sales_margin_flat,
            'AIRBAGS'                       AS airbags,
            'MISC CHARGES'                  AS misc_charges,
            'HARBOR MAINTENANCE FEE'        AS harbor_maint_fee,
            'REPALLETIZATION'               AS repalletization,
            'AIR FREIGHT'                   AS air_freight,
            'REPACKING'                     AS repacking,
            'CUSTOMS RECIPROCAL FEES'       AS customs_reciprocal_fees,
            'CUSTOMS - PENALTIES'           AS customs_penalties,
            'RECIPROCAL TARRIFFS'           AS reciprocal_tarriffs,
            'RECIPROCAL TARIFFS'            AS reciprocal_tariffs
        )
    )
),

allocation_base AS (
    SELECT
        m.*,
        t1.settlement_run_idx AS t1_run_idx,
        t1.t1_amt,
        t2.settlement_run_idx AS t2_run_idx,
        t2.t2_amt,
        t3.settlement_run_idx AS t3_run_idx,
        t3.t3_amt,
        lst.settlement_run_idx AS lot_run_idx,
        lst.lot_sales_amt,
        tp.t4_remaining_amt
    FROM pivoted_receipts m
    LEFT JOIN tier1_sales t1
      ON m.lot_index = t1.lot_index
     AND m.commodity = t1.commodity
     AND m.variety = t1.variety
     AND m.style = t1.style
     AND m.sizename = t1.sizename
     AND (
            m.color = t1.color
         OR (m.color IS NULL AND t1.color IS NULL)
     )
     AND (
            m.grade = t1.grade
         OR (m.grade IS NULL AND t1.grade IS NULL)
     )
    LEFT JOIN tier2_sales t2
      ON m.lot_index = t2.lot_index
     AND m.commodity = t2.commodity
     AND m.variety = t2.variety
    LEFT JOIN tier3_sales t3
      ON m.lot_index = t3.lot_index
    LEFT JOIN lot_sales_total lst
      ON m.lot_index = lst.lot_index
    LEFT JOIN tier_pool_totals_by_lot tp
      ON m.lot_index = tp.lot_index
),

allocation_calc AS (
    SELECT
        ab.*,
        CASE
            WHEN ab.t1_amt IS NOT NULL THEN 1
            ELSE 0
        END AS has_t1,
        CASE
            WHEN ab.t1_amt IS NULL
             AND ab.t2_amt IS NOT NULL THEN 1
            ELSE 0
        END AS has_t2_only,
        CASE
            WHEN ab.t1_amt IS NULL
             AND ab.t2_amt IS NULL
             AND ab.t3_amt IS NOT NULL THEN 1
            ELSE 0
        END AS has_t3_only,
        CASE
            WHEN ab.t1_amt IS NULL
             AND ab.t2_amt IS NULL
             AND ab.t3_amt IS NULL
             AND ab.lot_sales_amt IS NOT NULL THEN 1
            ELSE 0
        END AS has_t4_only,

        SUM(
            CASE
                WHEN ab.t1_amt IS NOT NULL
                THEN ab.pallet_full_attr_ratio
                ELSE 0
            END
        ) OVER (
            PARTITION BY
                ab.lot_index,
                ab.commodity,
                ab.variety,
                ab.style,
                ab.sizename,
                ab.color,
                ab.grade
        ) AS t1_ratio_sum,

        SUM(
            CASE
                WHEN ab.t1_amt IS NULL
                 AND ab.t2_amt IS NOT NULL
                THEN ab.pallet_variety_ratio
                ELSE 0
            END
        ) OVER (
            PARTITION BY
                ab.lot_index,
                ab.commodity,
                ab.variety
        ) AS t2_ratio_sum,

        SUM(
            CASE
                WHEN ab.t1_amt IS NULL
                 AND ab.t2_amt IS NULL
                 AND ab.t3_amt IS NOT NULL
                THEN ab.pallet_lot_fallback_ratio
                ELSE 0
            END
        ) OVER (
            PARTITION BY ab.lot_index
        ) AS t3_ratio_sum,

        SUM(
            CASE
                WHEN ab.t1_amt IS NULL
                 AND ab.t2_amt IS NULL
                 AND ab.t3_amt IS NULL
                 AND ab.lot_sales_amt IS NOT NULL
                THEN ab.pallet_lot_ratio
                ELSE 0
            END
        ) OVER (
            PARTITION BY ab.lot_index
        ) AS t4_ratio_sum
    FROM allocation_base ab
),

allocation_final AS (
    SELECT
        ac.*,
        CASE
            WHEN ac.has_t1 = 1
             AND NVL(ac.t1_ratio_sum, 0) <> 0
            THEN ac.pallet_full_attr_ratio / ac.t1_ratio_sum * ac.t1_amt

            WHEN ac.has_t2_only = 1
             AND NVL(ac.t2_ratio_sum, 0) <> 0
            THEN ac.pallet_variety_ratio / ac.t2_ratio_sum * ac.t2_amt

            WHEN ac.has_t3_only = 1
             AND NVL(ac.t3_ratio_sum, 0) <> 0
            THEN ac.pallet_lot_fallback_ratio / ac.t3_ratio_sum * ac.t3_amt

            WHEN ac.has_t4_only = 1
             AND NVL(ac.t4_ratio_sum, 0) <> 0
            THEN ac.pallet_lot_ratio / ac.t4_ratio_sum * ac.t4_remaining_amt

            ELSE 0
        END AS gross_sales_unrounded,

        CASE
            WHEN ac.has_t1 = 1 THEN ac.t1_run_idx
            WHEN ac.has_t2_only = 1 THEN ac.t2_run_idx
            WHEN ac.has_t3_only = 1 THEN ac.t3_run_idx
            WHEN ac.has_t4_only = 1 THEN ac.lot_run_idx
            ELSE ac.lot_run_idx
        END AS settlement_run
    FROM allocation_calc ac
),

final_calc AS (
    SELECT
        af.*,
        NVL(af.reciprocal_tarriffs, 0) + NVL(af.reciprocal_tariffs, 0) AS reciprocal_tariff_total,

        ROUND(af.pallet_lot_ratio * af.total_lot_charges_pool, 2) AS expected_allocated_charges,

        (
            NVL(af.cold_storage, 0)
          + NVL(af.customs, 0)
          + NVL(af.customs_exams, 0)
          + NVL(af.demurrage, 0)
          + NVL(af.freight_charges, 0)
          + NVL(af.grower_advance, 0)
          + NVL(af.inland_freight, 0)
          + NVL(af.inspection_charges, 0)
          + NVL(af.ocean_freight, 0)
          + NVL(af.qc_assessment, 0)
          + NVL(af.sales_margin, 0)
          + NVL(af.temp_recorder, 0)
          + NVL(af.grower_pmt_on_acct, 0)
          + NVL(af.fumigation, 0)
          + NVL(af.cold_treatment, 0)
          + NVL(af.grower_allow_adjust, 0)
          + NVL(af.dump_repack_fees, 0)
          + NVL(af.seal, 0)
          + NVL(af.freight_handling, 0)
          + NVL(af.customer_rebate, 0)
          + NVL(af.customer_allowance, 0)
          + NVL(af.sales_margin_flat, 0)
          + NVL(af.airbags, 0)
          + NVL(af.misc_charges, 0)
          + NVL(af.harbor_maint_fee, 0)
          + NVL(af.repalletization, 0)
          + NVL(af.air_freight, 0)
          + NVL(af.repacking, 0)
          + NVL(af.customs_reciprocal_fees, 0)
          + NVL(af.customs_penalties, 0)
          + NVL(af.reciprocal_tarriffs, 0)
          + NVL(af.reciprocal_tariffs, 0)
        ) AS mapped_allocated_charges
    FROM allocation_final af
)

SELECT
    fc.record_type,
    fc.lot_id,
    fc.lot_descr,
    fc.grower_name,
    fc.reference,
    fc.settlement_run,
    fc.lot_close_date,
    fc.received_date,
    fc.block_name,
    fc.pallet_tag_id,
    fc.commodity,
    fc.variety,
    fc.style,
    fc.sizename,
    fc.color,
    fc.grade,
    fc.qty_received,
    fc.pallet_lot_ratio,

    CASE
        WHEN fc.has_t1 = 1 THEN 'TIER 1 (Exact SKU w/ Grade)'
        WHEN fc.has_t2_only = 1 THEN 'TIER 2 (Variety Fallback)'
        WHEN fc.has_t3_only = 1 THEN 'TIER 3 (Lot Fallback)'
        WHEN fc.has_t4_only = 1 THEN 'TIER 4 (Lot Sales Fallback)'
        ELSE 'TIER 4 (Lot Sales Fallback)'
    END AS allocation_tier_applied,

    ROUND(fc.gross_sales_unrounded, 2) AS gross_sales_amount,

    fc.total_lot_charges_pool,
    fc.cold_storage,
    fc.customs,
    fc.customs_exams,
    fc.demurrage,
    fc.freight_charges,
    fc.grower_advance,
    fc.inland_freight,
    fc.inspection_charges,
    fc.ocean_freight,
    fc.qc_assessment,
    fc.sales_margin,
    fc.temp_recorder,
    fc.grower_pmt_on_acct,
    fc.fumigation,
    fc.cold_treatment,
    fc.grower_allow_adjust,
    fc.dump_repack_fees,
    fc.seal,
    fc.freight_handling,
    fc.customer_rebate,
    fc.customer_allowance,
    fc.sales_margin_flat,
    fc.airbags,
    fc.misc_charges,
    fc.harbor_maint_fee,
    fc.repalletization,
    fc.air_freight,
    fc.repacking,
    fc.customs_reciprocal_fees,
    fc.customs_penalties,
    fc.reciprocal_tariff_total AS reciprocal_tarriffs,

    CASE
        WHEN NVL(tr.has_tariff, 0) = 1
         AND lm.lot_total_qty > 0
         AND (lm.lot_total_sale - lm.lot_total_charges + lm.lot_total_advances) > 0
        THEN ROUND(
            fc.pallet_lot_ratio
            * ROUND(
                (lm.lot_total_sale - lm.lot_total_charges + lm.lot_total_advances) * 0.10,
                2
            ),
            2
        )
        ELSE 0
    END AS tariff,

    CASE
        WHEN ABS(fc.expected_allocated_charges - fc.mapped_allocated_charges) < 0.05
        THEN 0
        ELSE fc.expected_allocated_charges - fc.mapped_allocated_charges
    END AS unmapped_charges_alert

FROM final_calc fc
/* -------------------------------------------------------------------------
   LotMetrics & TariffRule joins match precisely with original code. 
------------------------------------------------------------------------- */
LEFT JOIN LotMetrics lm
  ON fc.lot_id = lm.lotid
LEFT JOIN TariffRule tr
  ON fc.settlement_run = tr.garunidx
"""

# ==========================================
# 1. CONFIGURATION MANAGER (JSON based)
# ==========================================
CONFIG_FILE = "settlement_config.json"
CACHE_FILE = "last_run_data.pkl"
AUTO_REFRESH_INTERVAL_MS = 15 * 60 * 1000

DEFAULT_CONFIG = {
    "UNIT_NAME": "Box",
    "REVENUE_COLUMNS": ["GROSS_SALES_AMOUNT"],
    "ADVANCE_COLUMNS": ["GROWER_ADVANCE", "GROWER_PMT_ON_ACCT"],
    "COST_MAPPINGS": {
        "Freight & Logistics": ["OCEAN_FREIGHT", "INLAND_FREIGHT", "FREIGHT_CHARGES", "FREIGHT_HANDLING", "DEMURRAGE", "AIR_FREIGHT"],
        "Customs & Fees": ["CUSTOMS", "CUSTOMS_EXAMS", "HARBOR_MAINT_FEE", "CUSTOMS_RECIPROCAL_FEES", "CUSTOMS_PENALTIES"],
        "Storage & Operations": ["COLD_STORAGE", "REPALLETIZATION", "DUMP_REPACK_FEES", "COLD_TREATMENT", "FUMIGATION", "TEMP_RECORDER", "SEAL", "AIRBAGS", "REPACKING"],
        "Inspections & Quality": ["INSPECTION_CHARGES", "QC_ASSESSMENT"],
        "Commissions": ["SALES_MARGIN", "SALES_MARGIN_FLAT"],
        "Adjustments & Other": ["GROWER_ALLOW_ADJUST", "CUSTOMER_REBATE", "MISC_CHARGES", "UNMAPPED_CHARGES_ALERT", "CUSTOMER_ALLOWANCE"]
    }
}

def load_config() -> dict:
    if not os.path.exists(CONFIG_FILE):
        with open(CONFIG_FILE, "w", encoding="utf-8") as f:
            json.dump(DEFAULT_CONFIG, f, indent=4)
        return DEFAULT_CONFIG
    try:
        with open(CONFIG_FILE, "r", encoding="utf-8") as f:
            cfg = json.load(f)
        needs_save = False
        for category, cols in DEFAULT_CONFIG["COST_MAPPINGS"].items():
            if category not in cfg.get("COST_MAPPINGS", {}):
                cfg["COST_MAPPINGS"][category] = cols
                needs_save = True
            else:
                for col in cols:
                    if col not in cfg["COST_MAPPINGS"][category]:
                        cfg["COST_MAPPINGS"][category].append(col)
                        needs_save = True
        if needs_save:
            with open(CONFIG_FILE, "w", encoding="utf-8") as f:
                json.dump(cfg, f, indent=4)
        return cfg
    except Exception:
        return DEFAULT_CONFIG

APP_CONFIG = load_config()
ALL_COST_COLUMNS = [cost for category in APP_CONFIG["COST_MAPPINGS"].values() for cost in category]

def build_required_columns() -> List[str]:
    cols = []
    cols.extend(APP_CONFIG["REVENUE_COLUMNS"])
    cols.extend(APP_CONFIG["ADVANCE_COLUMNS"])
    cols.extend(ALL_COST_COLUMNS)
    cols.extend([
        "QTY_RECEIVED", "TARIFF", "GROWER_NAME", "COMMODITY", "VARIETY",
        "STYLE", "SIZENAME", "COLOR", "GRADE", "LOT_ID", "BLOCK_NAME", 
        "PALLET_TAG_ID", "SETTLEMENT_RUN"
    ])
    return list(dict.fromkeys(cols))

REQUIRED_COLUMNS = build_required_columns()
TEXT_COLUMNS = ["GROWER_NAME", "COMMODITY", "VARIETY", "STYLE", "SIZENAME", "COLOR", "GRADE", "LOT_ID", "BLOCK_NAME", "PALLET_TAG_ID"]
INVALID_TEXT_VALUES = {"nan", "<na>", "none", "unknown", ""}
WHOLE_NUMBER_COLUMN_TOKENS = ("QTY", "QUANTITY", "VOLUME", "UNIT", "UNITS", "COUNT", "TOTAL BOXES")
WHOLE_NUMBER_COLUMN_NAMES = {"COUNT", "TOTAL GROWERS", "TOTAL LOTS", "TOTAL COMMODITIES", "TOTAL VARIETIES"}
PERCENT_COLUMN_NAMES = {"GROWER RETURN %", "GROWER RET %", "COMMISSION %", "VARIANCE %", "PERCENTILE RANK", "IMPORTANCE SCORE"}

def normalize_display_column_name(column_name: str) -> str:
    return str(column_name).replace("_", " ").strip().upper()

def is_run_column(column_name: str) -> bool:
    normalized = normalize_display_column_name(column_name)
    return "RUN" in normalized and "%" not in normalized

def is_whole_number_column(column_name: str) -> bool:
    normalized = normalize_display_column_name(column_name)
    return normalized in WHOLE_NUMBER_COLUMN_NAMES or any(token in normalized for token in WHOLE_NUMBER_COLUMN_TOKENS)

def is_percent_column(column_name: str) -> bool:
    normalized = normalize_display_column_name(column_name)
    return normalized in PERCENT_COLUMN_NAMES

def is_excel_percent_column(column_name: str) -> bool:
    normalized = normalize_display_column_name(column_name)
    return normalized in PERCENT_COLUMN_NAMES

# ==========================================
# 2. DATA CALCULATION ENGINE
# ==========================================
class SettlementEngine:
    def __init__(self):
        self.raw_df = pd.DataFrame()
        self.dynamic_full_df = pd.DataFrame()
        self.filtered_df = pd.DataFrame()
        self.filter_cache: Dict[str, List[str]] = {}
        self.analysis_cache: Dict[Tuple, object] = {}
        self.current_page_cache: Dict[str, pd.DataFrame] = {}
        self.view_cache_token: Tuple = ()
        self.last_error: str = ""
        self.inc_adv = True
        self.inc_tar = True

        self.adv_cols = APP_CONFIG.get("ADVANCE_COLUMNS", [])
        self.comm_cols = APP_CONFIG["COST_MAPPINGS"].get("Commissions", [])
        self.tar_cols = ["TARIFF"]
        
        self.ignore_cols = ["RECIPROCAL_TARRIFFS", "RECIPROCAL_TARIFFS"]
        
        self.opex_cols_map = {}
        for cat, cols in APP_CONFIG["COST_MAPPINGS"].items():
            if cat != "Commissions":
                for c in cols:
                    if c not in self.adv_cols and c not in self.tar_cols and c not in self.ignore_cols:
                        self.opex_cols_map[c] = cat

    def _read_sql_fast(self) -> pd.DataFrame:
        conn_str = "DSN=FAMOUSODBC;UID=Company_1_Rpt;PWD=FAMOUS"
        try:
            with pyodbc.connect(conn_str) as conn:
                with warnings.catch_warnings():
                    warnings.simplefilter("ignore", UserWarning)
                    
                    chunks = pd.read_sql(SQL_QUERY, conn, chunksize=100000)
                    processed_chunks = []
                    
                    for chunk in chunks:
                        chunk.columns = [str(c).strip().upper() for c in chunk.columns]
                        
                        keep = [c for c in REQUIRED_COLUMNS if c in chunk.columns]
                        if keep: 
                            chunk = chunk[keep]
                        
                        text_cols = [col for col in TEXT_COLUMNS if col in chunk.columns]
                        if text_cols:
                            chunk[text_cols] = chunk[text_cols].astype('category')

                        float_cols = chunk.select_dtypes(include=['float64']).columns.tolist()
                        if float_cols:
                            chunk[float_cols] = chunk[float_cols].astype('float32')
                            
                        processed_chunks.append(chunk)

                    if not processed_chunks:
                        raise ValueError("The SQL query returned no data.")
                        
                    df = pd.concat(processed_chunks, ignore_index=True)
                    
        except Exception as e:
            raise ValueError(f"Database connection or query failed:\n{e}")

        if "LOT_ID" not in df.columns and "GROWER_NAME" not in df.columns:
            raise ValueError("Required columns (like LOT_ID or GROWER_NAME) were not found.")
            
        return df

    @staticmethod
    def _ensure_columns(df: pd.DataFrame) -> pd.DataFrame:
        for col in REQUIRED_COLUMNS:
            if col not in df.columns:
                df[col] = "Unknown" if col in TEXT_COLUMNS else 0.0
        return df

    @staticmethod
    def _coerce_numeric(df: pd.DataFrame, cols: List[str]) -> None:
        for col in cols:
            if col in df.columns:
                df[col] = pd.to_numeric(df[col], errors="coerce").fillna(0.0)

    @staticmethod
    def _normalize_text(df: pd.DataFrame, cols: List[str]) -> None:
        for col in cols:
            if col in df.columns:
                cleaned = df[col].astype("string").str.strip()
                invalid_mask = cleaned.isna()
                lowered = cleaned.str.lower()
                invalid_mask |= lowered.isin(INVALID_TEXT_VALUES)
                df[col] = cleaned.mask(invalid_mask, "Unknown").astype(str)
            else:
                df[col] = "Unknown"

    @staticmethod
    def _safe_sorted_unique(series: pd.Series) -> List[str]:
        cleaned = pd.Series(series, copy=False).astype("string").str.strip()
        cleaned = cleaned[cleaned.notna()]
        if cleaned.empty:
            return []
        cleaned = cleaned[~cleaned.str.lower().isin(INVALID_TEXT_VALUES)]
        return sorted(pd.unique(cleaned).tolist())

    @staticmethod
    def _build_mask(df: pd.DataFrame, filter_dict: dict, search_text: str = "") -> pd.Series:
        mask = pd.Series(True, index=df.index)
        for col, val in filter_dict.items():
            if val != "All" and col in df.columns:
                mask &= df[col].eq(val)
        if search_text and "SEARCH_STRING" in df.columns:
            mask &= df["SEARCH_STRING"].str.contains(search_text.lower(), na=False, regex=False)
        return mask

    def _clear_analysis_cache(self) -> None:
        self.analysis_cache = {}

    def _clear_page_cache(self) -> None:
        self.current_page_cache = {}

    @staticmethod
    def _cache_result(value):
        return value

    def _get_cached_result(self, key: Tuple, builder):
        if key not in self.analysis_cache:
            self.analysis_cache[key] = self._cache_result(builder())
        cached = self.analysis_cache[key]
        return cached.copy(deep=False) if isinstance(cached, pd.DataFrame) else cached

    def _grouped_sum(self, df: pd.DataFrame, group_by_cols: List[str], cols_to_sum: List[str]) -> pd.DataFrame:
        if df.empty or not group_by_cols:
            return pd.DataFrame()
        return df.groupby(group_by_cols, as_index=False, observed=True)[cols_to_sum].sum()

    @staticmethod
    def _safe_divide(numerator: pd.Series, denominator: pd.Series) -> pd.Series:
        num = numerator.to_numpy(dtype=float, copy=False)
        den = denominator.to_numpy(dtype=float, copy=False)
        return pd.Series(
            np.divide(num, den, out=np.zeros_like(num, dtype=float), where=den != 0),
            index=numerator.index,
        )

    def _ensure_runtime_columns(self, df: pd.DataFrame) -> pd.DataFrame:
        opex_cols_map = getattr(self, "opex_cols_map", {})
        tar_cols = getattr(self, "tar_cols", [])
        adv_cols = getattr(self, "adv_cols", [])
        comm_cols = getattr(self, "comm_cols", [])
        if "SETTLEMENT_RUN" not in df.columns:
            df["SETTLEMENT_RUN"] = pd.NA

        run_series = df["SETTLEMENT_RUN"].astype("string").str.strip()
        run_numeric = pd.to_numeric(run_series, errors="coerce")
        df["SETTLEMENT_STATUS"] = np.where(run_numeric.notna(), "Settled", "Pending")
        normalized_runs = run_series.mask(run_series.isna() | run_series.str.lower().isin(INVALID_TEXT_VALUES), "Unknown")
        numeric_mask = run_numeric.notna()
        normalized_runs.loc[numeric_mask] = run_numeric.loc[numeric_mask].astype("Int64").astype(str)
        df["SETTLEMENT_RUN"] = normalized_runs.astype(str)
        df["RUN_STR"] = df["SETTLEMENT_RUN"]

        if "LOT_ID" in df.columns:
            lot_clean = df["LOT_ID"].astype(str).str.upper().str.rstrip("*")
            df["Region"] = np.select([lot_clean.str.endswith("W"), lot_clean.str.endswith("E")], ["West", "East"], default="Unknown")

        revenue_cols = [c for c in APP_CONFIG["REVENUE_COLUMNS"] if c in df.columns]
        df["TOTAL_REVENUE"] = df[revenue_cols].sum(axis=1) if revenue_cols else 0.0
        df["_OPEX_BASE"] = df[[c for c in opex_cols_map.keys() if c in df.columns]].sum(axis=1) if opex_cols_map else 0.0
        df["_TARIFFS_BASE"] = df[[c for c in tar_cols if c in df.columns]].sum(axis=1) if tar_cols else 0.0
        df["_ADVANCES_BASE"] = df[[c for c in adv_cols if c in df.columns]].sum(axis=1) if adv_cols else 0.0
        df["_COMMISSIONS_BASE"] = df[[c for c in comm_cols if c in df.columns]].sum(axis=1) if comm_cols else 0.0
        df["SEARCH_STRING"] = df[TEXT_COLUMNS].astype(str).agg(" ".join, axis=1).str.lower()
        return df

    def load_cache(self) -> bool:
        if os.path.exists(CACHE_FILE):
            try:
                self.raw_df = pd.read_pickle(CACHE_FILE)
                self.raw_df = self._ensure_runtime_columns(self.raw_df)
                self._clear_analysis_cache()
                self._clear_page_cache()
                self._build_filter_cache()
                self.apply_filters({}, "", True, True)
                return True
            except Exception as e:
                print(f"Failed to load cache: {e}")
                return False
        return False

    def save_cache(self, df: Optional[pd.DataFrame] = None):
        try:
            target_df = self.raw_df if df is None else df
            target_df.to_pickle(CACHE_FILE)
        except Exception as e:
            print(f"Failed to save cache: {e}")

    def _prepare_loaded_data(self) -> pd.DataFrame:
        df = self._read_sql_fast()
        df = self._ensure_columns(df)
        numeric_cols = APP_CONFIG["REVENUE_COLUMNS"] + ALL_COST_COLUMNS + APP_CONFIG["ADVANCE_COLUMNS"] + ["QTY_RECEIVED", "TARIFF"]
        self._coerce_numeric(df, numeric_cols)
        self._normalize_text(df, TEXT_COLUMNS)
        return self._ensure_runtime_columns(df)

    def apply_loaded_data(self, df: pd.DataFrame) -> None:
        self.raw_df = df
        self._clear_analysis_cache()
        self._clear_page_cache()
        self._build_filter_cache()
        self.last_error = ""

    def load_data(self) -> Tuple[bool, str, Optional[pd.DataFrame]]:
        try:
            df = self._prepare_loaded_data()
            self.save_cache(df)
            return True, "Data loaded successfully.", df
        except ValueError as ve:
            self.last_error = str(ve)
            return False, self.last_error, None
        except Exception:
            self.last_error = traceback.format_exc()
            return False, f"Unexpected error:\n{self.last_error}", None

    def _build_filter_cache(self) -> None:
        if self.raw_df.empty:
            self.filter_cache = {}
            return
        def safe_unique(col: str) -> List[str]:
            if col not in self.raw_df.columns:
                return []
            return self._safe_sorted_unique(self.raw_df[col])
        
        self.filter_cache = {
            "SETTLEMENT_STATUS": safe_unique("SETTLEMENT_STATUS"), "Region": safe_unique("Region"),
            "GROWER_NAME": safe_unique("GROWER_NAME"), "COMMODITY": safe_unique("COMMODITY"),
            "VARIETY": safe_unique("VARIETY"), "STYLE": safe_unique("STYLE"),
            "SIZENAME": safe_unique("SIZENAME"), "COLOR": safe_unique("COLOR"),
            "GRADE": safe_unique("GRADE"), "RUN_STR": safe_unique("RUN_STR"), "LOT_ID": safe_unique("LOT_ID"),
        }

    def apply_filters(self, filter_dict: dict, search_text: str = "", inc_adv: bool = True, inc_tar: bool = True):
        if self.raw_df.empty:
            self.filtered_df = pd.DataFrame()
            self.dynamic_full_df = pd.DataFrame()
            self._clear_analysis_cache()
            self._clear_page_cache()
            return

        self.inc_adv = inc_adv
        self.inc_tar = inc_tar
        df_full = self.raw_df

        df_full["_OPEX"] = df_full["_OPEX_BASE"] if "_OPEX_BASE" in df_full.columns else 0.0
        df_full["_TARIFFS"] = df_full["_TARIFFS_BASE"] if "_TARIFFS_BASE" in df_full.columns else 0.0
        df_full["_ADVANCES"] = df_full["_ADVANCES_BASE"] if "_ADVANCES_BASE" in df_full.columns else 0.0
        df_full["_COMMISSIONS"] = df_full["_COMMISSIONS_BASE"] if "_COMMISSIONS_BASE" in df_full.columns else 0.0

        live_tariffs = df_full["_TARIFFS"] if self.inc_tar else 0.0
        live_advances = df_full["_ADVANCES"] if self.inc_adv else 0.0

        df_full["TOTAL_COSTS"] = df_full["_OPEX"] + df_full["_COMMISSIONS"] + live_tariffs + live_advances
        df_full["NET_RETURN"] = df_full["TOTAL_REVENUE"] - df_full["TOTAL_COSTS"]
        
        qty = df_full["QTY_RECEIVED"].to_numpy(dtype=float, copy=False)
        net_return = df_full["NET_RETURN"].to_numpy(dtype=float, copy=False)
        df_full["RETURN_PER_BOX"] = np.divide(net_return, qty, out=np.zeros_like(qty, dtype=float), where=qty != 0)

        self.dynamic_full_df = df_full
        self.filtered_df = df_full.loc[self._build_mask(df_full, filter_dict, search_text)].copy()
        self.view_cache_token = (
            tuple(sorted((k, v) for k, v in filter_dict.items() if v != "All")),
            search_text.lower(),
            self.inc_adv,
            self.inc_tar,
        )
        self._clear_analysis_cache()
        self._clear_page_cache()

    def get_kpis(self) -> dict:
        if self.filtered_df.empty:
            return {"Total Boxes": 0, "Gross Sales": 0, "Total Costs": 0, "Net Return": 0, "Grower Return %": 0, "Cost / Box": 0, "Return / Box": 0, "Return / Box (No Tariff)": 0, "Operating Expenses": 0, "Total Advances": 0, "Total Tariffs": 0, "Avg Lot Return": 0, "Avg Grower Return": 0, "Total Growers": 0, "Total Lots": 0, "Total Commodities": 0, "Total Varieties": 0, "Commission Rev": 0, "Commission %": 0, "Comm / Box": 0, "Sales / FOB": 0}
        def build_kpis() -> dict:
            df = self.filtered_df
            qty = float(df["QTY_RECEIVED"].sum())
            revenue = float(df["TOTAL_REVENUE"].sum())
            costs = float(df["TOTAL_COSTS"].sum())
            net = float(df["NET_RETURN"].sum())
            op_ex = float(df["_OPEX"].sum())
            commissions = float(df["_COMMISSIONS"].sum())
            display_advances = float(df["_ADVANCES"].sum())
            display_tariff = float(df["_TARIFFS"].sum())
            applied_tariff = display_tariff if self.inc_tar else 0.0

            lot_returns = df.groupby("LOT_ID", observed=True)["NET_RETURN"].sum()
            grower_returns = df.groupby("GROWER_NAME", observed=True)["NET_RETURN"].sum()

            return {
                "Total Boxes": qty, "Gross Sales": revenue, "Total Costs": costs, "Net Return": net,
                "Grower Return %": (net / revenue if revenue != 0 else 0.0) * 100,
                "Cost / Box": (costs / qty if qty != 0 else 0.0),
                "Return / Box": (net / qty if qty != 0 else 0.0),
                "Return / Box (No Tariff)": ((net + applied_tariff) / qty if qty != 0 else 0.0),
                "Operating Expenses": op_ex, "Total Advances": display_advances, "Total Tariffs": display_tariff,
                "Avg Lot Return": float(lot_returns.mean()) if not lot_returns.empty else 0.0,
                "Avg Grower Return": float(grower_returns.mean()) if not grower_returns.empty else 0.0,
                "Total Growers": df["GROWER_NAME"].nunique(), "Total Lots": df["LOT_ID"].nunique(),
                "Total Commodities": df["COMMODITY"].nunique(), "Total Varieties": df["VARIETY"].nunique(),
                "Commission Rev": commissions, "Commission %": (commissions / revenue if revenue != 0 else 0.0) * 100, "Comm / Box": (commissions / qty if qty != 0 else 0.0),
                "Sales / FOB": (revenue / qty if qty != 0 else 0.0)
            }

        return self._get_cached_result(("kpis", self.inc_adv, self.inc_tar), build_kpis)

    def get_profitability_by(self, group_by_cols: list) -> pd.DataFrame:
        if self.filtered_df.empty: return pd.DataFrame()
        group_by_cols = [c for c in group_by_cols if c in self.filtered_df.columns]
        if not group_by_cols: return pd.DataFrame()
        def build_profitability() -> pd.DataFrame:
            cols_to_sum = [c for c in ["QTY_RECEIVED", "TOTAL_REVENUE", "TOTAL_COSTS", "_TARIFFS", "NET_RETURN", "_COMMISSIONS"] if c in self.filtered_df.columns]
            grouped = self._grouped_sum(self.filtered_df, group_by_cols, cols_to_sum)

            for col in ["QTY_RECEIVED", "TOTAL_REVENUE", "TOTAL_COSTS", "_TARIFFS", "NET_RETURN", "_COMMISSIONS"]:
                if col not in grouped.columns:
                    grouped[col] = 0.0

            qty = grouped["QTY_RECEIVED"]
            revenue = grouped["TOTAL_REVENUE"]
            grouped["Grower_Ret_%"] = self._safe_divide(grouped["NET_RETURN"], revenue) * 100
            grouped["Cost_Per_Box"] = self._safe_divide(grouped["TOTAL_COSTS"], qty)
            grouped["Return_Per_Box"] = self._safe_divide(grouped["NET_RETURN"], qty)
            grouped["Return_No_Tariff"] = self._safe_divide(grouped["NET_RETURN"] + grouped["_TARIFFS"], qty)
            grouped["Commission_%"] = self._safe_divide(grouped["_COMMISSIONS"], revenue) * 100
            grouped["Comm_Per_Box"] = self._safe_divide(grouped["_COMMISSIONS"], qty)

            grouped = grouped.rename(columns={"QTY_RECEIVED": "Qty", "TOTAL_REVENUE": "Gross_Sales", "TOTAL_COSTS": "Total_Costs", "NET_RETURN": "Net_Return", "_TARIFFS": "Tariff"})
            grouped = grouped.drop(columns=["_COMMISSIONS"], errors="ignore")
            return grouped.sort_values(by="Net_Return", ascending=False)

        return self._get_cached_result(("profitability", tuple(group_by_cols), self.inc_adv, self.inc_tar), build_profitability)

    def get_tag_analysis(self) -> pd.DataFrame:
        if self.filtered_df.empty: return pd.DataFrame()
        group_cols = [c for c in ["SETTLEMENT_RUN", "LOT_ID", "GROWER_NAME", "COMMODITY", "VARIETY", "BLOCK_NAME", "PALLET_TAG_ID"] if c in self.filtered_df.columns]
        
        cost_cols_found = [c for c in list(self.opex_cols_map.keys()) + self.tar_cols + self.adv_cols + self.comm_cols if c in self.filtered_df.columns]
        if "_TARIFFS" in self.filtered_df.columns:
            dynamic_cost_cols = ["_TARIFFS"]
        elif "TARIFF" in self.filtered_df.columns:
            dynamic_cost_cols = ["TARIFF"]
        else:
            dynamic_cost_cols = []
        cols_to_sum = list(dict.fromkeys(
            c for c in ["QTY_RECEIVED", "TOTAL_REVENUE", "TOTAL_COSTS", "NET_RETURN", "_COMMISSIONS"] + dynamic_cost_cols + [c for c in cost_cols_found if c != "TARIFF"]
            if c in self.filtered_df.columns
        ))

        def build_tag_analysis() -> pd.DataFrame:
            grouped = self._grouped_sum(self.filtered_df, group_cols, cols_to_sum)
            grouped = grouped.rename(columns={"QTY_RECEIVED": "Qty", "TOTAL_REVENUE": "Gross_Sales", "TOTAL_COSTS": "Total_Costs", "NET_RETURN": "Net_Return"})
            if "_TARIFFS" in grouped.columns:
                grouped["Tariff"] = grouped["_TARIFFS"]
            elif "TARIFF" in grouped.columns:
                grouped["Tariff"] = grouped["TARIFF"]
            else:
                grouped["Tariff"] = 0.0

            qty = grouped["Qty"] if "Qty" in grouped.columns else pd.Series(dtype=float)
            sales = grouped["Gross_Sales"] if "Gross_Sales" in grouped.columns else pd.Series(dtype=float)
            costs = grouped["Total_Costs"] if "Total_Costs" in grouped.columns else pd.Series(dtype=float)
            net = grouped["Net_Return"] if "Net_Return" in grouped.columns else pd.Series(dtype=float)
            comm = grouped["_COMMISSIONS"] if "_COMMISSIONS" in grouped.columns else pd.Series(dtype=float)

            grouped["Grower_Ret_%"] = self._safe_divide(net, sales) * 100
            grouped["Cost_Per_Box"] = self._safe_divide(costs, qty)
            grouped["Return_Per_Box"] = self._safe_divide(net, qty)
            grouped["Commission_Rev"] = comm
            grouped["Commission_%"] = self._safe_divide(comm, sales) * 100
            grouped["Comm_Per_Box"] = self._safe_divide(comm, qty)

            cost_cols_cleaned = [c for c in cost_cols_found if c != "TARIFF"]
            final_cols = group_cols + ["Qty", "Gross_Sales", "Total_Costs", "Tariff", "Net_Return", "Grower_Ret_%", "Cost_Per_Box", "Return_Per_Box", "Commission_Rev", "Commission_%", "Comm_Per_Box"] + cost_cols_cleaned
            final_cols = [c for c in final_cols if c in grouped.columns]

            if "LOT_ID" in grouped.columns and "PALLET_TAG_ID" in grouped.columns:
                return grouped[final_cols].sort_values(by=["LOT_ID", "PALLET_TAG_ID"])
            return grouped[final_cols]

        return self._get_cached_result(("tag_analysis", self.inc_adv, self.inc_tar), build_tag_analysis)

    def get_cost_breakdown(self, active_only=False) -> pd.DataFrame:
        if self.filtered_df.empty: return pd.DataFrame()
        def build_cost_breakdown() -> pd.DataFrame:
            qty = float(self.filtered_df["QTY_RECEIVED"].sum())
            cost_data = []

            for col, category in self.opex_cols_map.items():
                if col in self.filtered_df.columns:
                    total_amount = float(self.filtered_df[col].sum())
                    if total_amount != 0:
                        cost_data.append({"Category": category, "Charge Name": col, "Total Units": qty, "Total Amount": total_amount, "Cost / Box": (total_amount / qty if qty != 0 else 0.0)})

            for col in self.comm_cols:
                if col in self.filtered_df.columns:
                    total_amount = float(self.filtered_df[col].sum())
                    if total_amount != 0:
                        cost_data.append({"Category": "Commissions", "Charge Name": col, "Total Units": qty, "Total Amount": total_amount, "Cost / Box": (total_amount / qty if qty != 0 else 0.0)})

            if not active_only or self.inc_tar:
                for col in self.tar_cols:
                    if col in self.filtered_df.columns:
                        total_amount = float(self.filtered_df[col].sum())
                        if total_amount != 0:
                            cost_data.append({"Category": "Tariffs", "Charge Name": col, "Total Units": qty, "Total Amount": total_amount, "Cost / Box": (total_amount / qty if qty != 0 else 0.0)})

            if not active_only or self.inc_adv:
                for col in self.adv_cols:
                    if col in self.filtered_df.columns:
                        total_amount = float(self.filtered_df[col].sum())
                        if total_amount != 0:
                            cost_data.append({"Category": "Advances", "Charge Name": col, "Total Units": qty, "Total Amount": total_amount, "Cost / Box": (total_amount / qty if qty != 0 else 0.0)})

            return pd.DataFrame(cost_data).sort_values(by="Total Amount", ascending=False) if cost_data else pd.DataFrame()

        return self._get_cached_result(("cost_breakdown", active_only, self.inc_adv, self.inc_tar), build_cost_breakdown)

    def get_lot_details(self, lot_id: str) -> dict:
        if not hasattr(self, 'dynamic_full_df') or self.dynamic_full_df.empty: return {}
        def build_lot_details() -> dict:
            df = self.dynamic_full_df[self.dynamic_full_df["LOT_ID"] == str(lot_id)]
            if df.empty:
                return {}

            qty = float(df["QTY_RECEIVED"].sum())
            revenue = float(df["TOTAL_REVENUE"].sum())

            sales_cols = [c for c in ["COMMODITY", "VARIETY", "STYLE", "SIZENAME", "GRADE"] if c in df.columns]
            sales_breakdown = []
            if sales_cols:
                grouped_sales = df.groupby(sales_cols, as_index=False, observed=True)[["QTY_RECEIVED", "TOTAL_REVENUE"]].sum()
                for row in grouped_sales.itertuples(index=False):
                    s_qty = float(row.QTY_RECEIVED)
                    s_rev = float(row.TOTAL_REVENUE)
                    if s_rev != 0 or s_qty != 0:
                        parts = [str(getattr(row, c)) for c in sales_cols if str(getattr(row, c)) not in ("Unknown", "", "nan", "None")]
                        descr = " | ".join(parts) if parts else "Gross Sales"
                        avg_price = s_rev / s_qty if s_qty != 0 else 0.0
                        sales_breakdown.append({"description": descr, "qty": s_qty, "amount": s_rev, "avg_price": avg_price})

            individual_costs = {}
            valid_cols = list(self.opex_cols_map.keys()) + self.comm_cols
            if self.inc_tar:
                valid_cols.extend(self.tar_cols)
            if self.inc_adv:
                valid_cols.extend(self.adv_cols)

            for col in valid_cols:
                if col in df.columns:
                    amt = float(df[col].sum())
                    if amt != 0:
                        individual_costs[col] = amt

            total_costs = float(df["TOTAL_COSTS"].sum()) if "TOTAL_COSTS" in df.columns else 0.0
            net = float(df["NET_RETURN"].sum()) if "NET_RETURN" in df.columns else 0.0
            commission = float(df["_COMMISSIONS"].sum()) if "_COMMISSIONS" in df.columns else 0.0

            return {
                "Grower": str(df["GROWER_NAME"].iloc[0]), "Commodity": str(df["COMMODITY"].iloc[0]),
                "Status": str(df["SETTLEMENT_STATUS"].iloc[0]), "Region": str(df["Region"].iloc[0]),
                "Qty": qty, "Gross Sales": revenue, "Sales Breakdown": sales_breakdown,
                "Individual Costs": individual_costs, "Total Costs": total_costs, "Net Return": net,
                "Grower Return %": (net / revenue if revenue != 0 else 0.0) * 100, "Return / Box": (net / qty if qty != 0 else 0.0),
                "Commission Rev": commission, "Commission %": (commission / revenue if revenue != 0 else 0.0) * 100, "Comm / Box": (commission / qty if qty != 0 else 0.0)
            }

        return self._get_cached_result(("lot_details", str(lot_id), self.inc_adv, self.inc_tar), build_lot_details)

    def get_grower_benchmarking(self) -> pd.DataFrame:
        df = self.get_profitability_by(["GROWER_NAME"])
        if df.empty: return df
        avg_ret = df["Return_Per_Box"].mean()
        df["Variance_vs_Avg"] = df["Return_Per_Box"] - avg_ret
        df["Percentile_Rank"] = df["Return_Per_Box"].rank(pct=True) * 100
        df["Tier"] = np.where(df["Percentile_Rank"] > 75, "Top 25%", np.where(df["Percentile_Rank"] > 50, "Top 50%", np.where(df["Percentile_Rank"] > 25, "3rd Quartile", "Bottom Quartile")))
        return df.sort_values("Return_Per_Box", ascending=False)

    def get_product_intelligence(self) -> pd.DataFrame: return self.get_profitability_by(["COMMODITY", "VARIETY", "STYLE", "SIZENAME", "COLOR", "GRADE"])
    def get_variety_performance(self) -> pd.DataFrame: return self.get_profitability_by(["COMMODITY", "VARIETY"])
    def get_attribute_analysis(self) -> pd.DataFrame: return self.get_profitability_by(["COMMODITY", "VARIETY", "GRADE"])

    def get_price_variance_analysis(self) -> pd.DataFrame:
        if self.filtered_df.empty: return pd.DataFrame()
        profile_cols = ["COMMODITY", "VARIETY", "STYLE", "SIZENAME", "COLOR", "GRADE"]
        valid_cols = [c for c in profile_cols if c in self.filtered_df.columns]
        if not valid_cols:
            return pd.DataFrame()
        def build_price_variance() -> pd.DataFrame:
            market = self._grouped_sum(self.dynamic_full_df, valid_cols, ["NET_RETURN", "QTY_RECEIVED"])
            market["Market_Avg_Return"] = self._safe_divide(market["NET_RETURN"], market["QTY_RECEIVED"])
            market = market[valid_cols + ["Market_Avg_Return"]]

            grower_group_cols = valid_cols + ["GROWER_NAME"]
            growers = self._grouped_sum(self.filtered_df, grower_group_cols, ["QTY_RECEIVED", "NET_RETURN"])
            growers = growers.rename(columns={"QTY_RECEIVED": "Qty"})
            growers["Grower_Return"] = self._safe_divide(growers["NET_RETURN"], growers["Qty"])

            merged = pd.merge(growers, market, on=valid_cols, how="left")
            merged["Variance_Dollar"] = merged["Grower_Return"] - merged["Market_Avg_Return"]
            merged["Variance_%"] = self._safe_divide(merged["Variance_Dollar"], merged["Market_Avg_Return"]) * 100
            return merged[merged["Qty"] > 0].sort_values("Variance_Dollar", ascending=False)

        return self._get_cached_result(("price_variance", tuple(valid_cols), self.inc_adv, self.inc_tar), build_price_variance)

    def get_market_benchmarks(self) -> pd.DataFrame:
        if not hasattr(self, 'dynamic_full_df') or self.dynamic_full_df.empty: return pd.DataFrame()
        valid_cols = [c for c in ["COMMODITY", "VARIETY", "GRADE"] if c in self.dynamic_full_df.columns]
        if not valid_cols:
            return pd.DataFrame()
        def calc_bmarks(x):
            rets = (x["NET_RETURN"] / x["QTY_RECEIVED"].replace(0, np.nan)).fillna(0).to_numpy()
            rets = rets[rets > 0]
            if len(rets) == 0: return pd.Series({"Median": 0, "Top_25%": 0, "Bottom_25%": 0})
            return pd.Series({"Median": np.median(rets), "Top_25%": np.percentile(rets, 75), "Bottom_25%": np.percentile(rets, 25)})
        return self._get_cached_result(
            ("market_benchmarks", tuple(valid_cols), self.inc_adv, self.inc_tar),
            lambda: self.dynamic_full_df.groupby(valid_cols, observed=True).apply(calc_bmarks).reset_index()
        )

    def get_outlier_analysis(self) -> pd.DataFrame:
        if self.filtered_df.empty: return pd.DataFrame()
        mean_ret = self.dynamic_full_df["RETURN_PER_BOX"].mean()
        std_ret = self.dynamic_full_df["RETURN_PER_BOX"].std()
        if std_ret == 0: return pd.DataFrame()
        df = self.filtered_df[self.filtered_df["QTY_RECEIVED"] > 0].copy()
        df["Z_Score"] = (df["RETURN_PER_BOX"] - mean_ret) / std_ret
        return df[(df["Z_Score"] > 2) | (df["Z_Score"] < -2)][["LOT_ID", "GROWER_NAME", "COMMODITY", "RETURN_PER_BOX", "Z_Score"]].sort_values("Z_Score", ascending=False)

    def get_profitability_drivers(self) -> pd.DataFrame:
        if not SKLEARN_AVAILABLE or self.filtered_df.empty: return pd.DataFrame([{"Message": "Install scikit-learn for ML Driver Analysis"}])
        df = self.filtered_df[self.filtered_df["QTY_RECEIVED"] > 0].copy()
        features = [c for c in ["COMMODITY", "VARIETY", "GRADE", "SIZENAME", "COLOR", "Region"] if c in df.columns]
        df = df[features + ["RETURN_PER_BOX"]].dropna()
        if len(df) < 50: return pd.DataFrame([{"Message": "Not enough data points for ML (Need >50 rows)"}])
        le = LabelEncoder()
        X = df[features].apply(lambda col: le.fit_transform(col.astype(str)))
        rf = RandomForestRegressor(n_estimators=50, random_state=42, max_depth=5)
        rf.fit(X, df["RETURN_PER_BOX"])
        return pd.DataFrame({"Driver": features, "Importance_Score": rf.feature_importances_ * 100}).sort_values("Importance_Score", ascending=False)

    def get_executive_scorecards(self) -> pd.DataFrame:
        bench = self.get_grower_benchmarking()
        if bench.empty: return bench
        def grade(pct):
            if pct >= 90: return "A+"
            if pct >= 80: return "A"
            if pct >= 70: return "B"
            if pct >= 50: return "C"
            return "D"
        bench["Scorecard_Grade"] = bench["Percentile_Rank"].apply(grade)
        return bench[["GROWER_NAME", "Qty", "Gross_Sales", "Return_Per_Box", "Percentile_Rank", "Scorecard_Grade"]]

    def get_opportunity_finder(self) -> pd.DataFrame:
        if self.filtered_df.empty: return pd.DataFrame()
        df = self.get_grower_benchmarking()
        if df.empty: return pd.DataFrame()
        df = df[df["Return_Per_Box"] > 0]
        df["Opportunity"] = np.where(df["Percentile_Rank"] > 80, "Hidden Winner", np.where(df["Percentile_Rank"] < 25, "Underpriced Review Required", "Average Performance"))
        return df[["GROWER_NAME", "Qty", "Return_Per_Box", "Opportunity"]].sort_values("Return_Per_Box")

    def get_grower_performance_scorecard(self) -> pd.DataFrame: return self.get_grower_benchmarking()

    def get_return_variance_explanation(self) -> pd.DataFrame:
        if self.filtered_df.empty: return pd.DataFrame()
        df = self.get_profitability_by(["LOT_ID", "GROWER_NAME", "COMMODITY"])
        if df.empty: return df
        def build_return_variance() -> pd.DataFrame:
            market = self._grouped_sum(self.dynamic_full_df, ["COMMODITY"], ["NET_RETURN", "QTY_RECEIVED"])
            market["Expected_Return"] = self._safe_divide(market["NET_RETURN"], market["QTY_RECEIVED"])
            merged = pd.merge(df, market[["COMMODITY", "Expected_Return"]], on="COMMODITY", how="left")
            merged["Variance_$"] = merged["Return_Per_Box"] - merged["Expected_Return"]
            merged["Variance_%"] = self._safe_divide(merged["Variance_$"], merged["Expected_Return"]) * 100
            return merged[["LOT_ID", "GROWER_NAME", "COMMODITY", "Expected_Return", "Return_Per_Box", "Variance_$", "Variance_%"]].sort_values("Variance_$", ascending=True)

        return self._get_cached_result(("return_variance", self.inc_adv, self.inc_tar), build_return_variance)

    def get_product_dna_analysis(self) -> pd.DataFrame: return self.get_product_intelligence()

# ==========================================
# THREADING FOR SQL LOAD (NON-BLOCKING UX)
# ==========================================
class DataLoaderThread(QThread):
    finished_signal = Signal(bool, str, object)
    def __init__(self, engine):
        super().__init__()
        self.engine = engine

    def run(self):
        success, msg, df = self.engine.load_data()
        self.finished_signal.emit(success, msg, df)

# ==========================================
# 3. UI COMPONENTS (POLISHED)
# ==========================================
class PandasModel(QAbstractTableModel):
    def __init__(self, data=pd.DataFrame(), parent=None):
        super().__init__(parent)
        self._df = data
        self._columns = []
        self._set_data(data)

    def _set_data(self, data: pd.DataFrame):
        self.beginResetModel()
        self._df = data
        if data is None or data.empty:
            self._columns = []
        else:
            self._columns = data.columns.tolist()
        self.endResetModel()
        
    def get_dataframe(self) -> pd.DataFrame:
        return self._df

    def rowCount(self, parent=QModelIndex()): return 0 if self._df is None else len(self._df.index)
    def columnCount(self, parent=QModelIndex()): return len(self._columns)

    def data(self, index, role=Qt.ItemDataRole.DisplayRole):
        if not index.isValid(): return None
        if index.row() >= self.rowCount() or index.column() >= self.columnCount():
            return None
        val = self._df.iat[index.row(), index.column()]

        if role == Qt.ItemDataRole.DisplayRole:
            if pd.isna(val) or val == "": return ""
            col_name = self._columns[index.column()]
            if is_run_column(col_name):
                try: return str(int(float(val)))
                except: return str(val)
            if is_whole_number_column(col_name):
                try: return f"{float(val):,.0f}"
                except: return str(val)
            if isinstance(val, (int, float, np.integer, np.floating)):
                if is_percent_column(col_name): 
                    return f"{float(val):,.2f}%"
                return f"{float(val):,.2f}"
            return str(val)

        elif role == Qt.ItemDataRole.EditRole: return 0.0 if pd.isna(val) or val == "" else val
        elif role == Qt.ItemDataRole.UserRole: return 0.0 if pd.isna(val) or val == "" else val
        
        elif role == Qt.ItemDataRole.TextAlignmentRole:
            return int(Qt.AlignmentFlag.AlignRight | Qt.AlignmentFlag.AlignVCenter) if isinstance(val, (int, float, np.integer, np.floating)) else int(Qt.AlignmentFlag.AlignLeft | Qt.AlignmentFlag.AlignVCenter)
        
        elif role == Qt.ItemDataRole.ForegroundRole:
            if isinstance(val, (int, float, np.integer, np.floating)) and val < 0:
                return QColor("#e74c3c")
                
        return None

    def headerData(self, section, orientation, role):
        if role == Qt.ItemDataRole.DisplayRole and orientation == Qt.Orientation.Horizontal:
            if 0 <= section < len(self._columns): return str(self._columns[section]).replace("_", " ")
        return None

def export_df_to_excel(df: pd.DataFrame, title: str, parent_widget: QWidget):
    if df.empty:
        QMessageBox.information(parent_widget, "Export", "No data to export.")
        return
    file_name, _ = QFileDialog.getSaveFileName(parent_widget, f"Export {title}", f"{title}_Export.xlsx", "Excel Files (*.xlsx)")
    if not file_name: return
    try:
        QApplication.setOverrideCursor(Qt.CursorShape.WaitCursor)
        export_df = df.copy()
        for col in export_df.columns:
            if is_excel_percent_column(col) and pd.api.types.is_numeric_dtype(export_df[col]):
                export_df[col] = export_df[col] / 100
        with pd.ExcelWriter(file_name, engine='openpyxl') as writer:
            export_df.to_excel(writer, index=False, sheet_name='Data')
            worksheet = writer.sheets['Data']
            worksheet.freeze_panes = "A2"
            for idx, col in enumerate(df.columns):
                display_series = df[col]
                series = export_df[col]
                content_len = display_series.head(1000).astype(str).map(len).max()
                content_len = int(content_len) if pd.notna(content_len) else 0
                max_len = max(content_len, len(str(display_series.name))) + 2
                
                # Dynamic column letter conversion to bypass >26 columns bug ('[' error)
                col_idx = idx + 1
                letter = ""
                while col_idx > 0:
                    col_idx, remainder = divmod(col_idx - 1, 26)
                    letter = chr(65 + remainder) + letter
                    
                worksheet.column_dimensions[letter].width = min(max_len, 50)
                if is_whole_number_column(col):
                    for cell in worksheet[letter][1:]:
                        cell.number_format = "#,##0"
                elif is_excel_percent_column(col):
                    for cell in worksheet[letter][1:]:
                        cell.number_format = "0.00%"
                elif pd.api.types.is_numeric_dtype(series):
                    for cell in worksheet[letter][1:]:
                        cell.number_format = "#,##0.00"
        QMessageBox.information(parent_widget, "Success", f"Data exported successfully to\n{file_name}")
    except Exception as e:
        QMessageBox.critical(parent_widget, "Export Error", f"Failed to export data:\n{str(e)}")
    finally:
        QApplication.restoreOverrideCursor()

class KPICard(QFrame):
    clicked = Signal(str)

    def __init__(self, title, color="#2ecc71"):
        super().__init__()
        self.title_text = title
        self.setFrameShape(QFrame.Shape.StyledPanel)
        self.setCursor(Qt.CursorShape.PointingHandCursor)
        self.setStyleSheet(f"QFrame {{ background-color: #ffffff; border-radius: 8px; border: 1px solid #e1e8ed; border-top: 4px solid {color}; }} QFrame:hover {{ background-color: #f8fafc; }}")
        layout = QVBoxLayout(self)
        layout.setContentsMargins(15, 10, 15, 10) 
        layout.setSpacing(4)
        
        self.title_label = QLabel(title)
        self.title_label.setStyleSheet("color: #64748b; font-size: 13px; font-weight: bold; border: none; text-transform: uppercase; background: transparent;")
        self.value_label = QLabel("--")
        self.value_label.setStyleSheet("color: #1e293b; font-size: 22px; font-weight: bold; border: none; background: transparent;")
        
        layout.addWidget(self.title_label)
        layout.addWidget(self.value_label)
        self.setFixedHeight(75) 

    def mouseReleaseEvent(self, event):
        if event.button() == Qt.MouseButton.LeftButton:
            self.clicked.emit(self.title_text)
        super().mouseReleaseEvent(event)

# --- DEDICATED DRILL-DOWN WINDOWS ---
class GenericAnalysisWindow(QDialog):
    def __init__(self, title, df_summary, df_raw, parent=None):
        super().__init__(parent)
        self.setWindowTitle(f"Analysis: {title}")
        self.resize(1000, 700)
        self.setStyleSheet("background-color: #f8f9fa;") 
        layout = QVBoxLayout(self)

        header_layout = QHBoxLayout()
        lbl = QLabel(f"<b>{title}</b>")
        lbl.setStyleSheet("font-size: 20px; color: #1e293b;")
        header_layout.addWidget(lbl)
        
        btn_export = QPushButton("Export Data")
        btn_export.setStyleSheet("background-color: #27ae60; color: white; font-weight: bold; padding: 6px 15px; border-radius: 4px;")
        btn_export.clicked.connect(lambda: export_df_to_excel(df_summary, title, self))
        header_layout.addWidget(btn_export, alignment=Qt.AlignmentFlag.AlignRight)
        
        btn_close = QPushButton("Close")
        btn_close.setStyleSheet("background-color: #95a5a6; color: white; padding: 6px 15px; border-radius: 4px;")
        btn_close.clicked.connect(self.accept)
        header_layout.addWidget(btn_close, alignment=Qt.AlignmentFlag.AlignRight)
        
        layout.addLayout(header_layout)

        if not df_raw.empty:
            kpi_layout = QHBoxLayout()
            qty = df_raw["QTY_RECEIVED"].sum() if "QTY_RECEIVED" in df_raw.columns else 0
            rev = df_raw["TOTAL_REVENUE"].sum() if "TOTAL_REVENUE" in df_raw.columns else 0
            net = df_raw["NET_RETURN"].sum() if "NET_RETURN" in df_raw.columns else 0
            
            c1 = KPICard("Total Boxes", "#3498db"); c1.value_label.setText(f"{qty:,.0f}")
            c2 = KPICard("Gross Sales", "#2ecc71"); c2.value_label.setText(f"${rev:,.2f}")
            c3 = KPICard("Net Return", "#9b59b6"); c3.value_label.setText(f"${net:,.2f}")
            c4 = KPICard("Return / Box", "#e67e22"); c4.value_label.setText(f"${(net/qty) if qty != 0 else 0:,.2f}")
            
            kpi_layout.addWidget(c1); kpi_layout.addWidget(c2); kpi_layout.addWidget(c3); kpi_layout.addWidget(c4)
            layout.addLayout(kpi_layout)

        lbl_det = QLabel("<b>Detailed Breakdown</b>")
        lbl_det.setStyleSheet("font-size: 14px; margin-top: 10px; color: #334155;")
        layout.addWidget(lbl_det)
        
        table = QTableView()
        table.setAlternatingRowColors(True)
        table.setStyleSheet("QTableView { background-color: white; alternate-background-color: #fbfbfb; border: 1px solid #e1e8ed; }")
        table.setSelectionBehavior(QTableView.SelectionBehavior.SelectRows)
        table.setEditTriggers(QTableView.EditTrigger.NoEditTriggers)
        table.setSortingEnabled(True)
        table.horizontalHeader().setSectionResizeMode(QHeaderView.ResizeMode.Interactive)
        proxy = QSortFilterProxyModel(table)
        proxy.setSortRole(Qt.ItemDataRole.UserRole)
        proxy.setSourceModel(PandasModel(df_summary, parent=table))
        table.setModel(proxy)
        
        if parent and hasattr(parent, 'handle_dialog_drilldown'):
            table.doubleClicked.connect(lambda idx, t=table: parent.handle_dialog_drilldown(idx, t))

        layout.addWidget(table)

class TagDetailDialog(QDialog):
    def __init__(self, tag_id, df_tag, parent=None):
        super().__init__(parent)
        self.setWindowTitle(f"Tag Detail: {tag_id}")
        self.resize(800, 600)
        self.setStyleSheet("background-color: #f8f9fa;")
        layout = QVBoxLayout(self)
        
        header = QHBoxLayout()
        lbl = QLabel(f"<b>Tag ID:</b> {tag_id}")
        lbl.setStyleSheet("font-size: 18px; color: #1e293b;")
        header.addWidget(lbl)
        btn_close = QPushButton("Close")
        btn_close.setStyleSheet("background-color: #95a5a6; color: white; padding: 6px 15px; border-radius: 4px;")
        btn_close.clicked.connect(self.accept)
        header.addWidget(btn_close, alignment=Qt.AlignmentFlag.AlignRight)
        layout.addLayout(header)

        table = QTableView()
        table.setAlternatingRowColors(True)
        table.setStyleSheet("QTableView { background-color: white; alternate-background-color: #fbfbfb; border: 1px solid #e1e8ed; }")
        proxy = QSortFilterProxyModel(table)
        proxy.setSortRole(Qt.ItemDataRole.UserRole)
        proxy.setSourceModel(PandasModel(df_tag, parent=table))
        table.setModel(proxy)
        layout.addWidget(table)

class LotDetailDialog(QDialog):
    def __init__(self, lot_id, details, parent=None):
        super().__init__(parent)
        self.setWindowTitle(f"Lot Profitability Detail: {lot_id}")
        self.resize(500, 750) 
        self.setStyleSheet("background-color: #f8f9fa;")
        layout = QVBoxLayout(self)
            
        header_text = f"<b>Lot:</b> {lot_id}<br><b>Region:</b> {details.get('Region','')}<br><b>Grower:</b> {details.get('Grower','')}<br><b>Commodity:</b> {details.get('Commodity','')} ({details.get('Status','')})<br><b>Boxes:</b> {details.get('Qty',0):,.0f}"
        header_label = QLabel(header_text)
        header_label.setStyleSheet("font-size: 14px; margin-bottom: 10px; color: #1e293b;")
        layout.addWidget(header_label)

        scroll_area = QScrollArea()
        scroll_area.setWidgetResizable(True)
        scroll_area.setStyleSheet("QScrollArea { border: 1px solid #e1e8ed; background-color: white; border-radius: 8px; }")
        scroll_widget = QWidget()
        scroll_widget.setStyleSheet("background-color: white;")
        form = QFormLayout(scroll_widget)
        form.setFieldGrowthPolicy(QFormLayout.FieldGrowthPolicy.AllNonFixedFieldsGrow)

        def add_row(label, value, is_bold=False, color="#334155", is_percent=False):
            lbl = QLabel(label)
            if isinstance(value, (int, float)):
                if is_percent: val_str = f"{value:,.2f}%"
                else:
                    if value < 0: val_str = f"-${abs(value):,.2f}"
                    else: val_str = f"${value:,.2f}"
                val = QLabel(val_str)
            else: val = QLabel(str(value))
            val.setAlignment(Qt.AlignmentFlag.AlignRight | Qt.AlignmentFlag.AlignVCenter)
            if is_bold:
                lbl.setStyleSheet(f"font-weight: bold; color: {color}; font-size: 14px; margin-top: 5px;")
                val.setStyleSheet(f"font-weight: bold; color: {color}; font-size: 14px; margin-top: 5px;")
            else:
                lbl.setStyleSheet("font-size: 13px; color: #475569;")
                if isinstance(value, (int, float)) and value < 0:
                    val.setStyleSheet("font-size: 13px; color: #e74c3c; font-weight: bold;") 
                else:
                    val.setStyleSheet("font-size: 13px; color: #475569;")
            form.addRow(lbl, val)

        add_row("Gross Sales:", details.get("Gross Sales", 0), True, "#2ecc71")
        
        sales_items = details.get("Sales Breakdown", [])
        if sales_items:
            for item in sales_items:
                desc = item["description"]
                q = item["qty"]
                avg_p = item["avg_price"]
                amt = item["amount"]
                label_str = f"  + {desc} ({q:,.0f} bxs @ ${avg_p:,.2f}/bx):"
                add_row(label_str, amt)
                
        form.addRow(QLabel(""), QLabel(""))
        for cost_name, amt in details.get("Individual Costs", {}).items(): 
            add_row(f"  - {str(cost_name).replace('_', ' ').title()}:", amt)
            
        add_row("Total Costs:", details.get("Total Costs",0), True, "#e74c3c")
        form.addRow(QLabel(""), QLabel(""))
        add_row("Grower Net Return:", details.get("Net Return",0), True, "#9b59b6")
        add_row("Grower Return %:", details.get("Grower Return %",0), True, "#2c3e50", is_percent=True)
        add_row("Return / Box:", details.get("Return / Box",0), True, "#2c3e50")
        form.addRow(QLabel(""), QLabel(""))
        add_row("Importer Commission Rev:", details.get("Commission Rev",0), True, "#3498db")
        add_row("Commission %:", details.get("Commission %",0), True, "#2980b9", is_percent=True)
        add_row("Commission / Box:", details.get("Comm / Box",0), True, "#1abc9c")

        scroll_area.setWidget(scroll_widget)
        layout.addWidget(scroll_area)
        
        btn_close = QPushButton("Close")
        btn_close.setStyleSheet("background-color: #95a5a6; color: white; padding: 8px 15px; border-radius: 4px; font-weight:bold;")
        btn_close.clicked.connect(self.accept)
        layout.addWidget(btn_close)

class GrowerPerformanceDetailDialog(QDialog):
    def __init__(self, grower_name, engine, parent=None):
        super().__init__(parent)
        self.setWindowTitle(f"Grower Scorecard: {grower_name}")
        self.resize(700, 600)
        self.setStyleSheet("background-color: #f8f9fa;")
        layout = QVBoxLayout(self)
        
        header = QHBoxLayout()
        lbl = QLabel(f"<b>{grower_name} Executive Summary</b>")
        lbl.setStyleSheet("font-size: 20px; color: #1e293b;")
        header.addWidget(lbl)
        
        btn_close = QPushButton("Close")
        btn_close.setStyleSheet("background-color: #95a5a6; color: white; padding: 6px 15px; border-radius: 4px;")
        btn_close.clicked.connect(self.accept)
        header.addWidget(btn_close, alignment=Qt.AlignmentFlag.AlignRight)
        layout.addLayout(header)

        df = engine.filtered_df[engine.filtered_df["GROWER_NAME"] == grower_name] if not engine.filtered_df.empty else pd.DataFrame()
        if df.empty:
            layout.addWidget(QLabel("No data available for this grower."))
            return

        qty = df["QTY_RECEIVED"].sum()
        rev = df["TOTAL_REVENUE"].sum()
        costs = df["TOTAL_COSTS"].sum()
        net = df["NET_RETURN"].sum()
        advances = df["_ADVANCES"].sum() if "_ADVANCES" in df.columns and engine.inc_adv else 0.0
        tariffs = df["_TARIFFS"].sum() if "_TARIFFS" in df.columns and engine.inc_tar else 0.0

        grid = QGridLayout()
        grid.addWidget(QLabel("<b>Total Boxes:</b>"), 0, 0); grid.addWidget(QLabel(f"{qty:,.0f}"), 0, 1)
        grid.addWidget(QLabel("<b>Gross Revenue:</b>"), 1, 0); grid.addWidget(QLabel(f"${rev:,.2f}"), 1, 1)
        grid.addWidget(QLabel("<b>Total Costs:</b>"), 2, 0); grid.addWidget(QLabel(f"${costs:,.2f}"), 2, 1)
        grid.addWidget(QLabel("  - Advances:"), 3, 0); grid.addWidget(QLabel(f"${advances:,.2f}"), 3, 1)
        grid.addWidget(QLabel("  - Tariffs:"), 4, 0); grid.addWidget(QLabel(f"${tariffs:,.2f}"), 4, 1)
        grid.addWidget(QLabel("<b>Grower Net Return:</b>"), 5, 0); grid.addWidget(QLabel(f"${net:,.2f}"), 5, 1)
        grid.addWidget(QLabel("<b>Grower Return %:</b>"), 6, 0); grid.addWidget(QLabel(f"{(net/rev*100) if rev != 0 else 0:.2f}%"), 6, 1)
        grid.addWidget(QLabel("<b>Return / Box:</b>"), 7, 0); grid.addWidget(QLabel(f"${(net/qty) if qty != 0 else 0:.2f}"), 7, 1)

        layout.addLayout(grid)
        
        lbl_mix = QLabel("<b>Commodity Mix</b>")
        lbl_mix.setStyleSheet("font-size: 14px; margin-top: 15px; color: #334155;")
        layout.addWidget(lbl_mix)
        
        mix_df = df.groupby("COMMODITY", as_index=False, observed=True)[["QTY_RECEIVED", "TOTAL_REVENUE", "NET_RETURN"]].sum()
        mix_df["Return/Box"] = (mix_df["NET_RETURN"] / mix_df["QTY_RECEIVED"].replace(0, np.nan)).fillna(0)
        mix_df = mix_df.sort_values("QTY_RECEIVED", ascending=False)
        
        table = QTableView()
        table.setStyleSheet("QTableView { background-color: white; alternate-background-color: #fbfbfb; border: 1px solid #e1e8ed; }")
        table.horizontalHeader().setSectionResizeMode(QHeaderView.ResizeMode.Stretch)
        table.setModel(PandasModel(mix_df, parent=table))
        layout.addWidget(table)

class VarianceExplanationDialog(QDialog):
    def __init__(self, lot_id, grower_name, commodity, engine, parent=None):
        super().__init__(parent)
        self.setWindowTitle("Variance Explanation")
        self.resize(550, 350)
        self.setStyleSheet("background-color: #f8f9fa;")
        layout = QVBoxLayout(self)
        
        lbl = QLabel(f"<b>Variance Analysis</b><br><span style='color:#64748b;'>Lot: {lot_id} | Grower: {grower_name} | {commodity}</span>")
        lbl.setStyleSheet("font-size: 16px; margin-bottom: 10px; color: #1e293b;")
        layout.addWidget(lbl)
        
        df_full = engine.dynamic_full_df
        if df_full.empty: return
        lot_df = df_full[df_full["LOT_ID"] == str(lot_id)]
        if lot_df.empty: return
        lot_ret = (lot_df["NET_RETURN"].sum() / lot_df["QTY_RECEIVED"].sum()) if lot_df["QTY_RECEIVED"].sum() != 0 else 0
        comm_df = df_full[df_full["COMMODITY"] == commodity]
        market_ret = (comm_df["NET_RETURN"].sum() / comm_df["QTY_RECEIVED"].sum()) if comm_df["QTY_RECEIVED"].sum() != 0 else 0
        
        diff = lot_ret - market_ret
        diff_pct = (diff / market_ret * 100) if market_ret != 0 else 0

        frame = QFrame()
        frame.setStyleSheet("background-color: white; border: 1px solid #e1e8ed; border-radius: 8px;")
        form_layout = QVBoxLayout(frame)
        form = QFormLayout()
        
        l1 = QLabel(f"${market_ret:,.2f}"); l1.setStyleSheet("font-size: 14px;")
        l2 = QLabel(f"${lot_ret:,.2f}"); l2.setStyleSheet("font-size: 14px; font-weight: bold;")
        
        form.addRow("<b>Expected Return (Market Avg):</b>", l1)
        form.addRow("<b>Actual Return (This Lot):</b>", l2)
        diff_lbl = QLabel(f"${diff:,.2f} ({diff_pct:,.1f}%)")
        diff_lbl.setStyleSheet(f"color: {'#2ecc71' if diff >= 0 else '#e74c3c'}; font-weight: bold; font-size: 16px;")
        form.addRow("<b>Difference:</b>", diff_lbl)
        
        form_layout.addLayout(form)
        layout.addWidget(frame)
        
        note = QLabel("<i>Note: Full impact driver breakdown requires additional ML modeling limits to be exceeded in production environments.</i>")
        note.setStyleSheet("color: #94a3b8;")
        note.setWordWrap(True)
        layout.addWidget(note)
        
        layout.addStretch()
        btn_close = QPushButton("Close")
        btn_close.setStyleSheet("background-color: #95a5a6; color: white; padding: 8px 15px; border-radius: 4px; font-weight: bold;")
        btn_close.clicked.connect(self.accept)
        layout.addWidget(btn_close)

class ChartDetailDialog(QDialog):
    def __init__(self, title, chart_view, df, parent=None):
        super().__init__(parent)
        self.setWindowTitle(f"Detailed Analysis: {title}")
        self.resize(1100, 750)
        self.setStyleSheet("background-color: #f8f9fa;")
        layout = QVBoxLayout(self)
        
        chart_view.setStyleSheet("background-color: white; border: 1px solid #e1e8ed; border-radius: 8px;")
        layout.addWidget(chart_view)
        
        lbl = QLabel("<b>Underlying Dataset</b>")
        lbl.setStyleSheet("font-size: 14px; margin-top: 10px; color: #1e293b;")
        layout.addWidget(lbl)
        
        table = QTableView()
        table.setAlternatingRowColors(True)
        table.setStyleSheet("QTableView { background-color: white; alternate-background-color: #fbfbfb; border: 1px solid #e1e8ed; }")
        table.horizontalHeader().setSectionResizeMode(QHeaderView.ResizeMode.Interactive)
        proxy = QSortFilterProxyModel(table)
        proxy.setSortRole(Qt.ItemDataRole.UserRole)
        proxy.setSourceModel(PandasModel(df, parent=table))
        table.setModel(proxy)
        layout.addWidget(table)
        
        btn_layout = QHBoxLayout()
        btn_export = QPushButton("Export Data")
        btn_export.setStyleSheet("background-color: #27ae60; color: white; padding: 6px 15px; border-radius: 4px; font-weight: bold;")
        btn_export.clicked.connect(lambda: export_df_to_excel(df, title, self))
        
        btn_close = QPushButton("Close")
        btn_close.setStyleSheet("background-color: #95a5a6; color: white; padding: 6px 15px; border-radius: 4px; font-weight: bold;")
        btn_close.clicked.connect(self.accept)
        
        btn_layout.addStretch()
        btn_layout.addWidget(btn_export)
        btn_layout.addWidget(btn_close)
        layout.addLayout(btn_layout)

class ClickableChartView(QChartView):
    def __init__(self, chart, title, parent=None):
        super().__init__(chart, parent)
        self.title = title
        self.main_window = parent
        self.df_cache = pd.DataFrame()
        
    def set_data(self, df):
        self.df_cache = df

    def mouseDoubleClickEvent(self, event):
        if not self.df_cache.empty and self.main_window:
            new_chart = QChart()
            new_chart.setTitle(self.chart().title())
            
            for series in self.chart().series():
                if isinstance(series, QBarSeries):
                    new_series = QBarSeries()
                    for barset in series.barSets():
                        new_set = QBarSet(barset.label())
                        for i in range(barset.count()): new_set.append(barset.at(i))
                        new_series.append(new_set)
                    new_chart.addSeries(new_series)
                    
                    if self.chart().axes(Qt.Orientation.Horizontal):
                        categories = self.chart().axes(Qt.Orientation.Horizontal)[0].categories()
                        axisX = QBarCategoryAxis()
                        axisX.append(categories)
                        new_chart.addAxis(axisX, Qt.AlignmentFlag.AlignBottom)
                        new_series.attachAxis(axisX)
                        
                        axisY = QValueAxis()
                        new_chart.addAxis(axisY, Qt.AlignmentFlag.AlignLeft)
                        new_series.attachAxis(axisY)
                        
                elif isinstance(series, QPieSeries):
                    new_series = QPieSeries()
                    for slice in series.slices(): new_series.append(slice.label(), slice.value())
                    new_series.setLabelsVisible(True)
                    new_chart.addSeries(new_series)
                    
                elif isinstance(series, QHorizontalBarSeries):
                    new_series = QHorizontalBarSeries()
                    for barset in series.barSets():
                        new_set = QBarSet(barset.label())
                        for i in range(barset.count()): new_set.append(barset.at(i))
                        new_series.append(new_set)
                    new_chart.addSeries(new_series)
                    
                    if self.chart().axes(Qt.Orientation.Vertical):
                        categories = self.chart().axes(Qt.Orientation.Vertical)[0].categories()
                        axisY = QBarCategoryAxis()
                        axisY.append(categories)
                        new_chart.addAxis(axisY, Qt.AlignmentFlag.AlignLeft)
                        new_series.attachAxis(axisY)
                        
                        axisX = QValueAxis()
                        new_chart.addAxis(axisX, Qt.AlignmentFlag.AlignBottom)
                        new_series.attachAxis(axisX)
                        
            cv = QChartView(new_chart)
            cv.setRenderHint(QPainter.RenderHint.Antialiasing)
            dialog = ChartDetailDialog(self.title, cv, self.df_cache, self.main_window)
            dialog.exec()
        super().mouseDoubleClickEvent(event)

# ==========================================
# 4. MAIN APPLICATION WINDOW
# ==========================================
class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Executive Profitability Intelligence Platform")
        self.setStyleSheet("QMainWindow { background-color: #f4f7f6; }")
        self.showMaximized()

        self.engine = SettlementEngine()
        self.updating_filters = False
        self.loader_thread: Optional[DataLoaderThread] = None
        self._load_preserve_state = False
        self._load_show_error_dialog = True
        self._pending_view_state: Optional[dict] = None
        
        self.root_stack = QStackedWidget()
        self.setCentralWidget(self.root_stack)

        self.setup_menu()
        self.setup_app_ui()
        self.setCentralWidget(self.app_widget)
        self.auto_refresh_timer = QTimer(self)
        self.auto_refresh_timer.setInterval(AUTO_REFRESH_INTERVAL_MS)
        self.auto_refresh_timer.timeout.connect(self.refresh_data_in_background)
        self.auto_refresh_timer.start()

        # STARTUP: Check for local cache before hitting the database
        if self.engine.load_cache():
            self.initialize_filters()
            try:
                mod_time = datetime.fromtimestamp(os.path.getmtime(CACHE_FILE)).strftime("%Y-%m-%d %I:%M:%S %p")
                self.lbl_timestamp.setText(f"Last Update (Local Cache): {mod_time}")
                self.statusBar().showMessage(f"Loaded local cache from {mod_time}. Refreshing in background...", 5000)
            except Exception:
                pass
            self.refresh_data_in_background()
        else:
            self.load_data_from_db()

    def setup_menu(self):
        menu = self.menuBar()
        menu.clear() 
        file_menu = menu.addMenu("File")
        load_action = QAction("Update Data from Database", self)
        load_action.triggered.connect(self.load_data_from_db)
        file_menu.addAction(load_action)

        tools_menu = menu.addMenu("Executive Tools")
        fav_action = QAction("Favorites Menu", self)
        fav_action.triggered.connect(lambda: QMessageBox.information(self, "Favorites", "Saved views and favorites will appear here."))
        tools_menu.addAction(fav_action)
        
        refresh_action = QAction("Force Refresh Analytics", self)
        refresh_action.triggered.connect(self.update_dashboard)
        tools_menu.addAction(refresh_action)

    def setup_app_ui(self):
        self.app_widget = QWidget()
        main_layout = QHBoxLayout(self.app_widget)

        # --- SIDEBAR WITH SCROLL AREA ---
        sidebar = QFrame()
        sidebar.setFixedWidth(250)
        sidebar.setStyleSheet("background-color: #1e293b; color: white; border-right: 1px solid #0f172a;")
        sidebar_main_layout = QVBoxLayout(sidebar)
        sidebar_main_layout.setContentsMargins(0, 0, 0, 0)
        sidebar_main_layout.setSpacing(0)

        title = QLabel("Intelligence\nPlatform")
        title.setFont(QFont("Arial", 16, QFont.Weight.Bold))
        title.setStyleSheet("padding: 20px 15px 10px 15px; color: #f8fafc; border-bottom: 1px solid #334155;")
        sidebar_main_layout.addWidget(title)

        nav_scroll = QScrollArea()
        nav_scroll.setWidgetResizable(True)
        nav_scroll.setFrameShape(QFrame.Shape.NoFrame)
        nav_scroll.setStyleSheet("""
            QScrollArea { background-color: transparent; border: none; }
            QScrollBar:vertical { background: #0f172a; width: 6px; }
            QScrollBar::handle:vertical { background: #334155; border-radius: 3px; }
            QScrollBar::add-line:vertical, QScrollBar::sub-line:vertical { height: 0px; }
        """)

        nav_widget = QWidget()
        nav_widget.setStyleSheet("background-color: transparent;")
        sidebar_layout = QVBoxLayout(nav_widget)
        sidebar_layout.setAlignment(Qt.AlignmentFlag.AlignTop)
        sidebar_layout.setContentsMargins(0, 10, 0, 10)
        sidebar_layout.setSpacing(2)

        self.pages_map = {
            "Dashboard": 0, "Executive Dashboard": 1,         
            "Grower Analysis": 2, "Grower Benchmarking": 3,         
            "Grower Performance Scorecard": 4, "Product Analysis": 5,
            "Product Intelligence": 6, "Product DNA Analysis": 7,         
            "Variety Performance": 8, "Attribute Analysis": 9,          
            "Price Variance Analysis": 10, "Return Variance Explanation": 11,
            "Market Benchmarking": 12, "Outlier Detection": 13,          
            "Opportunity Finder": 14, "Profitability Drivers": 15,       
            "Executive Scorecards": 16, "Lot Analysis": 17,
            "Tag Analysis": 18, "Cost Analysis": 19
        }
        self.page_names_by_index = {idx: page_name for page_name, idx in self.pages_map.items()}

        self.stacked_widget = QStackedWidget()
        self.stacked_widget.currentChanged.connect(self.on_page_changed)
        self.tables = {}
        self.nav_buttons = {}

        for page_name, idx in self.pages_map.items():
            btn = QPushButton(page_name)
            btn.setMinimumHeight(38)
            btn.setCursor(Qt.CursorShape.PointingHandCursor)
            btn.setStyleSheet("QPushButton { text-align: left; padding: 10px 10px 10px 15px; font-size: 13px; background-color: transparent; color: #cbd5e1; border: none; } QPushButton:hover { background-color: #334155; color: white; }")
            btn.clicked.connect(lambda checked, i=idx: self.navigate_to_page(i))
            self.nav_buttons[idx] = btn
            sidebar_layout.addWidget(btn)

            page = QWidget()
            layout = QVBoxLayout(page)
            layout.setContentsMargins(0, 0, 0, 0)
            
            if page_name == "Dashboard": self.setup_original_dashboard(layout)
            elif page_name == "Executive Dashboard": self.setup_executive_dashboard(layout)
            else:
                header_layout = QHBoxLayout()
                lbl = QLabel(f"<b>{page_name}</b>")
                lbl.setStyleSheet("font-size: 20px; color: #1e293b;")
                header_layout.addWidget(lbl)
                
                btn_export = QPushButton("Export...")
                btn_export.setStyleSheet("background-color: #27ae60; color: white; padding: 6px 15px; font-weight: bold; border-radius: 4px;")
                menu = QMenu(btn_export)
                menu.addAction("Export Current View").triggered.connect(lambda checked, pn=page_name: self.export_table_to_excel(pn))
                btn_export.setMenu(menu)
                header_layout.addWidget(btn_export, alignment=Qt.AlignmentFlag.AlignRight)
                layout.addLayout(header_layout)
                
                table = QTableView()
                table.setAlternatingRowColors(True)
                table.setStyleSheet("QTableView { background-color: white; alternate-background-color: #fbfbfb; border: 1px solid #e2e8f0; border-radius: 4px; }")
                table.setSelectionBehavior(QTableView.SelectionBehavior.SelectRows)
                table.setEditTriggers(QTableView.EditTrigger.NoEditTriggers)
                table.setSortingEnabled(True)
                
                table.setContextMenuPolicy(Qt.ContextMenuPolicy.CustomContextMenu)
                table.customContextMenuRequested.connect(lambda pos, tbl=table: self.show_context_menu(pos, tbl))
                table.doubleClicked.connect(lambda idx, tbl=table: self.handle_dialog_drilldown(idx, tbl))

                header = table.horizontalHeader()
                header.setSectionResizeMode(QHeaderView.ResizeMode.Interactive)
                header.setDefaultSectionSize(110)
                header.setStretchLastSection(True)

                layout.addWidget(table)
                self.tables[page_name] = table

            self.stacked_widget.addWidget(page)

        sidebar_layout.addStretch()
        
        btn_update_sidebar = QPushButton("🔄 Fetch from DB")
        btn_update_sidebar.setStyleSheet("QPushButton { text-align: left; padding: 10px 10px 10px 15px; font-size: 13px; background-color: #2563eb; color: white; border: none; font-weight: bold; border-radius: 4px; margin: 10px 10px 0px 10px; } QPushButton:hover { background-color: #1d4ed8; }")
        btn_update_sidebar.setCursor(Qt.CursorShape.PointingHandCursor)
        btn_update_sidebar.clicked.connect(self.load_data_from_db)
        sidebar_layout.addWidget(btn_update_sidebar)

        self.lbl_timestamp = QLabel("Last Update: Never")
        self.lbl_timestamp.setStyleSheet("color: #94a3b8; font-size: 11px; padding: 0px 0px 10px 15px;")
        sidebar_layout.addWidget(self.lbl_timestamp)

        nav_scroll.setWidget(nav_widget)
        sidebar_main_layout.addWidget(nav_scroll)
        
        self.sidebar = sidebar
        main_layout.addWidget(self.sidebar)

        self.navigate_to_page(0)
        self.on_page_changed(0)

        # --- RIGHT CONTENT AREA ---
        content_layout = QVBoxLayout()
        content_layout.setContentsMargins(15, 15, 15, 15)
        content_layout.setSpacing(10)

        # FILTER PANEL
        self.filter_group = QGroupBox("Strategic Filters")
        self.filter_group.setStyleSheet("""
            QGroupBox { font-weight: bold; border: 1px solid #cbd5e1; border-radius: 8px; margin-top: 1ex; background-color: white; padding: 15px 10px 10px 10px; } 
            QGroupBox::title { subcontrol-origin: margin; left: 15px; padding: 0 5px 0 5px; color: #475569; }
            QLabel { color: #334155; font-weight: 600; }
        """)
        
        filter_layout = QVBoxLayout()
        self.combos = {}

        def create_combo(name, min_w=100):
            cb = QComboBox()
            cb.addItem("All")
            cb.setMinimumWidth(min_w)
            cb.setSizePolicy(QSizePolicy.Policy.Expanding, QSizePolicy.Policy.Fixed)
            cb.setStyleSheet("QComboBox { padding: 5px; border: 1px solid #cbd5e1; border-radius: 4px; background-color: #ffffff; color: #0f172a; min-height: 24px; } QComboBox::drop-down { border-left: 1px solid #cbd5e1; } QComboBox QAbstractItemView { background-color: #ffffff; color: #0f172a; selection-background-color: #3b82f6; selection-color: white; }")
            self.combos[name] = cb
            return cb

        grid = QGridLayout()
        grid.setHorizontalSpacing(15)
        grid.setVerticalSpacing(12)
        
        # R0: Primary Filters
        grid.addWidget(QLabel("Grower:"), 0, 0)
        c_grower = create_combo("GROWER_NAME", 200)
        grid.addWidget(c_grower, 0, 1, 1, 3) 
        
        grid.addWidget(QLabel("Search:"), 0, 4)
        self.search_box = QLineEdit()
        self.search_box.setPlaceholderText("Search Lot, Tag, Commodity...")
        self.search_box.setStyleSheet("QLineEdit { padding: 5px; border: 1px solid #cbd5e1; border-radius: 4px; background-color: #ffffff; color: #0f172a; min-height: 24px; }")
        self.search_box.setSizePolicy(QSizePolicy.Policy.Expanding, QSizePolicy.Policy.Fixed)
        grid.addWidget(self.search_box, 0, 5, 1, 3) 
        
        grid.addWidget(QLabel("Status:"), 0, 8)
        grid.addWidget(create_combo("SETTLEMENT_STATUS", 100), 0, 9)

        # R1: Secondary Filters
        grid.addWidget(QLabel("Commodity:"), 1, 0); grid.addWidget(create_combo("COMMODITY", 100), 1, 1)
        grid.addWidget(QLabel("Variety:"), 1, 2);   grid.addWidget(create_combo("VARIETY", 100), 1, 3)
        grid.addWidget(QLabel("Region:"), 1, 4);    grid.addWidget(create_combo("Region", 100), 1, 5)
        grid.addWidget(QLabel("Grade:"), 1, 6);     grid.addWidget(create_combo("GRADE", 100), 1, 7)
        grid.addWidget(QLabel("Style:"), 1, 8);     grid.addWidget(create_combo("STYLE", 100), 1, 9)

        # R2: Tertiary Filters
        grid.addWidget(QLabel("Lot ID:"), 2, 0);    grid.addWidget(create_combo("LOT_ID", 100), 2, 1)
        grid.addWidget(QLabel("Run ID:"), 2, 2);    grid.addWidget(create_combo("RUN_STR", 100), 2, 3)
        grid.addWidget(QLabel("Size:"), 2, 4);      grid.addWidget(create_combo("SIZENAME", 100), 2, 5)
        grid.addWidget(QLabel("Color:"), 2, 6);     grid.addWidget(create_combo("COLOR", 100), 2, 7)
        
        for col in [1, 3, 5, 7, 9]:
            grid.setColumnStretch(col, 1)

        filter_layout.addLayout(grid)
        
        # Divider Line
        line = QFrame()
        line.setFrameShape(QFrame.Shape.HLine)
        line.setStyleSheet("background-color: #cbd5e1; margin-top: 5px; margin-bottom: 5px;")
        filter_layout.addWidget(line)

        # R3: Control Toggles
        r4 = QHBoxLayout()
        self.cb_inc_advances = QCheckBox("Include Advances"); self.cb_inc_advances.setChecked(True)
        self.cb_inc_advances.setStyleSheet("color: #334155; font-weight: bold;")
        r4.addWidget(self.cb_inc_advances)
        
        self.cb_inc_tariffs = QCheckBox("Include Tariffs"); self.cb_inc_tariffs.setChecked(True)
        self.cb_inc_tariffs.setStyleSheet("color: #334155; font-weight: bold;")
        r4.addWidget(self.cb_inc_tariffs)
        
        lbl_limit = QLabel("Chart Limit:")
        lbl_limit.setStyleSheet("color: #334155; font-weight: bold; margin-left: 15px;")
        r4.addWidget(lbl_limit)
        
        self.cb_top_n = QComboBox()
        self.cb_top_n.addItems(["Top 5", "Top 10", "Top 20", "All"])
        self.cb_top_n.setCurrentText("Top 10") 
        self.cb_top_n.setStyleSheet("QComboBox { padding: 4px; border: 1px solid #cbd5e1; border-radius: 4px; font-weight: bold; color: #0f172a; }")
        r4.addWidget(self.cb_top_n)
        
        r4.addStretch()
        
        self.btn_reset_filters = QPushButton("Reset Filters")
        self.btn_reset_filters.setStyleSheet("background-color: #e74c3c; color: white; padding: 6px 20px; font-weight: bold; border-radius: 4px;")
        self.btn_reset_filters.setCursor(Qt.CursorShape.PointingHandCursor)
        self.btn_reset_filters.clicked.connect(self.reset_filters)
        r4.addWidget(self.btn_reset_filters)

        filter_layout.addLayout(r4)
        self.filter_group.setLayout(filter_layout)
        
        for cb in self.combos.values(): cb.currentTextChanged.connect(self.on_filter_changed)
        self.search_box.textChanged.connect(self.on_filter_changed)
        self.cb_inc_advances.stateChanged.connect(self.on_filter_changed)
        self.cb_inc_tariffs.stateChanged.connect(self.on_filter_changed)
        self.cb_top_n.currentTextChanged.connect(self.update_charts)

        # Top Control Bar for UI Toggles (Sidebar + Filter Panel)
        top_control_layout = QHBoxLayout()
        
        self.btn_toggle_sidebar = QPushButton("◀ Hide Sidebar")
        self.btn_toggle_sidebar.setStyleSheet("background: transparent; color: #2563eb; font-weight: bold; text-align: left;")
        self.btn_toggle_sidebar.setCursor(Qt.CursorShape.PointingHandCursor)
        self.btn_toggle_sidebar.clicked.connect(self.toggle_sidebar)

        self.btn_collapse = QPushButton("▼ Hide Filters")
        self.btn_collapse.setStyleSheet("background: transparent; color: #3b82f6; font-weight: bold; text-align: right;")
        self.btn_collapse.setCursor(Qt.CursorShape.PointingHandCursor)
        self.btn_collapse.clicked.connect(self.toggle_filters)

        top_control_layout.addWidget(self.btn_toggle_sidebar)
        top_control_layout.addStretch()
        top_control_layout.addWidget(self.btn_collapse)

        content_layout.addLayout(top_control_layout)
        content_layout.addWidget(self.filter_group)
        content_layout.addWidget(self.stacked_widget)
        main_layout.addLayout(content_layout)

    def toggle_sidebar(self):
        if self.sidebar.isVisible():
            self.sidebar.setVisible(False)
            self.btn_toggle_sidebar.setText("▶ Show Sidebar")
        else:
            self.sidebar.setVisible(True)
            self.btn_toggle_sidebar.setText("◀ Hide Sidebar")

    def toggle_filters(self):
        if self.filter_group.isVisible():
            self.filter_group.setVisible(False)
            self.btn_collapse.setText("▶ Show Filters")
        else:
            self.filter_group.setVisible(True)
            self.btn_collapse.setText("▼ Hide Filters")

    def show_context_menu(self, pos, table):
        index = table.indexAt(pos)
        if not index.isValid(): return
        
        menu = QMenu()
        menu.setStyleSheet("QMenu { background-color: white; border: 1px solid #ccc; } QMenu::item:selected { background-color: #e0f2fe; color: black; }")
        drill_action = menu.addAction("🔍 Analyze / Drill-Down")
        
        action = menu.exec(table.viewport().mapToGlobal(pos))
        if action == drill_action:
            self.handle_dialog_drilldown(index, table)

    def navigate_to_page(self, index: int):
        self.stacked_widget.setCurrentIndex(index)

    def on_page_changed(self, index: int):
        for i, btn in self.nav_buttons.items():
            if i == index:
                btn.setStyleSheet("QPushButton { text-align: left; padding: 10px 10px 10px 15px; font-size: 14px; background-color: #3b82f6; color: white; border: none; font-weight: bold; border-left: 5px solid #60a5fa; }")
            else:
                btn.setStyleSheet("QPushButton { text-align: left; padding: 10px 10px 10px 15px; font-size: 13px; background-color: transparent; color: #cbd5e1; border: none; } QPushButton:hover { background-color: #334155; color: white; }")
        if not self.engine.raw_df.empty and index not in (0, 1):
            self.refresh_current_page_table(index)

    def setup_original_dashboard(self, layout):
        kpi_row1, kpi_row2, kpi_row3 = QHBoxLayout(), QHBoxLayout(), QHBoxLayout()
        self.kpi_qty = KPICard("Total Boxes", "#3498db"); self.kpi_qty.clicked.connect(self.handle_kpi_card_click)
        self.kpi_sales = KPICard("Gross Sales", "#2ecc71"); self.kpi_sales.clicked.connect(self.handle_kpi_card_click)
        self.kpi_net = KPICard("Grower Net Return", "#9b59b6"); self.kpi_net.clicked.connect(self.handle_kpi_card_click)
        
        self.kpi_grower_ret = KPICard("Grower Return %", "#8e44ad"); self.kpi_grower_ret.clicked.connect(self.handle_kpi_card_click)
        self.kpi_return = KPICard("Return / Box", "#e67e22"); self.kpi_return.clicked.connect(self.handle_kpi_card_click)
        self.kpi_return_notariff = KPICard("Return / Box (No Tariff)", "#e74c3c"); self.kpi_return_notariff.clicked.connect(self.handle_kpi_card_click)
        
        self.kpi_comm_rev = KPICard("Commission Rev", "#34495e"); self.kpi_comm_rev.clicked.connect(self.handle_kpi_card_click)
        self.kpi_comm_pct = KPICard("Commission %", "#2c3e50"); self.kpi_comm_pct.clicked.connect(self.handle_kpi_card_click)
        self.kpi_comm_pb = KPICard("Comm / Box", "#7f8c8d"); self.kpi_comm_pb.clicked.connect(self.handle_kpi_card_click)
        
        kpi_row1.addWidget(self.kpi_qty); kpi_row1.addWidget(self.kpi_sales); kpi_row1.addWidget(self.kpi_net)
        kpi_row2.addWidget(self.kpi_grower_ret); kpi_row2.addWidget(self.kpi_return); kpi_row2.addWidget(self.kpi_return_notariff)
        kpi_row3.addWidget(self.kpi_comm_rev); kpi_row3.addWidget(self.kpi_comm_pct); kpi_row3.addWidget(self.kpi_comm_pb)
        
        layout.addLayout(kpi_row1); layout.addLayout(kpi_row2); layout.addLayout(kpi_row3); layout.addStretch()

    def setup_executive_dashboard(self, layout):
        scroll = QScrollArea()
        scroll.setWidgetResizable(True)
        scroll.setFrameShape(QFrame.Shape.NoFrame)
        scroll.setStyleSheet("background-color: transparent;")
        container = QWidget()
        container.setStyleSheet("background-color: transparent;")
        v_layout = QVBoxLayout(container)

        grid = QGridLayout()
        
        lbl_grower = QLabel("GROWER ECONOMICS")
        lbl_grower.setStyleSheet("font-size: 15px; font-weight: bold; color: #475569; padding-top: 5px;")
        grid.addWidget(lbl_grower, 0, 0, 1, 6)

        # EXACT RE-ORDER REQUESTED BY USER
        self.kpi_ex_sales = KPICard("Gross Sales", "#2ecc71"); self.kpi_ex_sales.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_sales, 1, 0)
        self.kpi_ex_exp = KPICard("Total Costs", "#e74c3c"); self.kpi_ex_exp.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_exp, 1, 1)
        self.kpi_ex_cpb = KPICard("Cost / Box", "#c0392b"); self.kpi_ex_cpb.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_cpb, 1, 2)
        self.kpi_ex_net = KPICard("Grower Net Return", "#9b59b6"); self.kpi_ex_net.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_net, 1, 3)
        self.kpi_ex_rpb = KPICard("Return / Box", "#27ae60"); self.kpi_ex_rpb.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_rpb, 1, 4)
        self.kpi_ex_rpb_notariff = KPICard("Return / Box (No Tariff)", "#8e44ad"); self.kpi_ex_rpb_notariff.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_rpb_notariff, 1, 5)

        lbl_importer = QLabel("IMPORTER ECONOMICS & OPERATIONS")
        lbl_importer.setStyleSheet("font-size: 15px; font-weight: bold; color: #475569; padding-top: 15px;")
        grid.addWidget(lbl_importer, 2, 0, 1, 6)

        self.kpi_ex_comm = KPICard("Commission Rev", "#3498db"); self.kpi_ex_comm.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_comm, 3, 0)
        self.kpi_ex_comm_pct = KPICard("Commission %", "#2980b9"); self.kpi_ex_comm_pct.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_comm_pct, 3, 1)
        self.kpi_ex_comm_pb = KPICard("Comm / Box", "#1abc9c"); self.kpi_ex_comm_pb.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_comm_pb, 3, 2)
        self.kpi_ex_adv = KPICard("Total Advances", "#f39c12"); self.kpi_ex_adv.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_adv, 3, 3)
        self.kpi_ex_tar = KPICard("Total Tariffs", "#d35400"); self.kpi_ex_tar.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_tar, 3, 4)
        self.kpi_ex_opex = KPICard("Operating Expenses", "#e67e22"); self.kpi_ex_opex.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_opex, 3, 5)
        
        self.kpi_ex_qty = KPICard("Total Boxes", "#3498db"); self.kpi_ex_qty.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_qty, 4, 0)
        self.kpi_ex_fob = KPICard("Sales / FOB", "#34495e"); self.kpi_ex_fob.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_fob, 4, 1)
        self.kpi_ex_tc = KPICard("Total Commodities", "#2c3e50"); self.kpi_ex_tc.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_tc, 4, 2)
        self.kpi_ex_tv = KPICard("Total Varieties", "#7f8c8d"); self.kpi_ex_tv.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_tv, 4, 3)
        self.kpi_ex_alr = KPICard("Avg Lot Return", "#16a085"); self.kpi_ex_alr.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_alr, 4, 4)
        self.kpi_ex_agr = KPICard("Avg Grower Return", "#1abc9c"); self.kpi_ex_agr.clicked.connect(self.handle_kpi_card_click); grid.addWidget(self.kpi_ex_agr, 4, 5)
        
        v_layout.addLayout(grid)

        chart_layout = QHBoxLayout()
        self.chart_commodity = QChart()
        self.chart_commodity.setTitle("Net Return by Commodity")
        self.chart_commodity.setBackgroundRoundness(8)
        self.view_commodity = ClickableChartView(self.chart_commodity, "Commodity Returns", self)
        self.view_commodity.setRenderHint(QPainter.RenderHint.Antialiasing)
        self.view_commodity.setMinimumHeight(350)
        self.view_commodity.setStyleSheet("background-color: transparent;")
        chart_layout.addWidget(self.view_commodity)
        
        self.chart_costs = QChart()
        self.chart_costs.setTitle("Cost Drivers Breakdown")
        self.chart_costs.setBackgroundRoundness(8)
        self.view_costs = ClickableChartView(self.chart_costs, "Cost Breakdown", self)
        self.view_costs.setRenderHint(QPainter.RenderHint.Antialiasing)
        self.view_costs.setMinimumHeight(350)
        self.view_costs.setStyleSheet("background-color: transparent;")
        chart_layout.addWidget(self.view_costs)
        
        v_layout.addLayout(chart_layout)
        scroll.setWidget(container)
        layout.addWidget(scroll)

    def handle_kpi_card_click(self, title: str):
        if self.engine.filtered_df.empty: return
        
        raw_df = self.engine.filtered_df
        df = pd.DataFrame()
        desc = ""

        if title == "Operating Expenses":
            desc = "Operating Expenses isolates pure logistical and operational costs (Excludes Tariffs, Commissions, and Advances)."
            cost_df = self.engine.get_cost_breakdown(active_only=False)
            if not cost_df.empty:
                df = cost_df[~cost_df["Category"].isin(["Commissions", "Tariffs", "Advances"])]
        
        elif title in ["Total Costs", "Cost / Box"]:
            desc = "Full cost breakdown. This view completely mirrors the toggles in your filter panel."
            df = self.engine.get_cost_breakdown(active_only=True)
            
        elif title == "Gross Sales":
            desc = "Gross Sales broken down by Commodity and Variety."
            df = self.engine.get_profitability_by(["COMMODITY", "VARIETY"])[["COMMODITY", "VARIETY", "Qty", "Gross_Sales"]]
            
        elif title in ["Grower Net Return", "Grower Return %", "Return / Box"]:
            desc = "Overall Grower profitability and benchmarking performance."
            df = self.engine.get_grower_benchmarking()[["GROWER_NAME", "Qty", "Gross_Sales", "Total_Costs", "Net_Return", "Return_Per_Box"]]
            
        elif title in ["Commission Rev", "Commission %", "Comm / Box"]:
            desc = "Importer commissions separated by Lot."
            df = self.engine.get_profitability_by(["LOT_ID", "GROWER_NAME"])[["LOT_ID", "GROWER_NAME", "Qty", "Gross_Sales", "Commission_Rev", "Commission_%", "Comm_Per_Box"]]
            
        elif title == "Total Advances":
            desc = "Absolute sum of all Grower Advances strictly isolated from standard operations."
            cost_df = self.engine.get_cost_breakdown(active_only=False)
            if not cost_df.empty:
                df = cost_df[cost_df["Category"] == "Advances"]
                
        elif title == "Total Tariffs":
            desc = "Absolute sum of the SQL TARIFF column strictly isolated from standard operations."
            cost_df = self.engine.get_cost_breakdown(active_only=False)
            if not cost_df.empty:
                df = cost_df[cost_df["Category"] == "Tariffs"]
                
        elif title in ["Total Boxes", "Avg Lot Return", "Avg Grower Return", "Return / Box (No Tariff)"]:
            desc = "High-level lot summary overview."
            df = raw_df[["LOT_ID", "GROWER_NAME", "COMMODITY", "QTY_RECEIVED", "TOTAL_REVENUE", "TOTAL_COSTS", "NET_RETURN"]]
            
        elif title == "Sales / FOB":
            desc = "Average FOB per unit broken down by Commodity and Variety."
            df = self.engine.get_profitability_by(["COMMODITY", "VARIETY"])[["COMMODITY", "VARIETY", "Qty", "Gross_Sales"]]
            df["Avg FOB"] = (df["Gross_Sales"] / df["Qty"].replace(0, np.nan)).fillna(0)
            df = df[["COMMODITY", "VARIETY", "Qty", "Gross_Sales", "Avg FOB"]]
            
        else:
            desc = f"Count summary for {title}."
            clean_title = title.replace("Total ", "").upper()
            if clean_title.endswith("S"): clean_title = clean_title[:-1]
            if clean_title == "VARIETIE": clean_title = "VARIETY"
            if clean_title == "COMMODITIE": clean_title = "COMMODITY"
            df = pd.DataFrame({"Metric": [title], "Count": [len(raw_df[clean_title].unique()) if clean_title in raw_df.columns else 0]})

        if df.empty:
            QMessageBox.information(self, "Drill Down", f"No detailed breakdown available for {title} based on current filters.")
            return

        dialog = QDialog(self)
        dialog.setWindowTitle(f"KPI Drill-Down: {title}")
        dialog.resize(800, 500)
        dialog.setStyleSheet("background-color: #f8f9fa;")
        layout = QVBoxLayout(dialog)

        header = QLabel(f"<b>{title} Breakdown</b><br><span style='color:#64748b; font-size:13px;'>{desc}</span>")
        header.setStyleSheet("font-size: 18px; color: #1e293b;")
        layout.addWidget(header)

        table = QTableView()
        table.setAlternatingRowColors(True)
        table.setStyleSheet("QTableView { background-color: white; alternate-background-color: #fbfbfb; border: 1px solid #e1e8ed; }")
        table.setSelectionBehavior(QTableView.SelectionBehavior.SelectRows)
        table.setEditTriggers(QTableView.EditTrigger.NoEditTriggers)
        table.setSortingEnabled(True)
        table.horizontalHeader().setSectionResizeMode(QHeaderView.ResizeMode.Interactive)
        proxy = QSortFilterProxyModel(table)
        proxy.setSortRole(Qt.ItemDataRole.UserRole)
        proxy.setSourceModel(PandasModel(df, parent=table))
        table.setModel(proxy)
        layout.addWidget(table)
        
        btn_layout = QHBoxLayout()
        btn_export = QPushButton("Export Data")
        btn_export.setStyleSheet("background-color: #27ae60; color: white; padding: 6px 15px; border-radius: 4px; font-weight: bold;")
        btn_export.clicked.connect(lambda: export_df_to_excel(df, title, dialog))
        
        btn_close = QPushButton("Close")
        btn_close.setStyleSheet("background-color: #95a5a6; color: white; padding: 6px 15px; border-radius: 4px; font-weight:bold;")
        btn_close.clicked.connect(dialog.accept)
        
        btn_layout.addStretch()
        btn_layout.addWidget(btn_export)
        btn_layout.addWidget(btn_close)
        layout.addLayout(btn_layout)

        dialog.exec()

    def _capture_view_state(self) -> dict:
        return {
            "filters": {col: cb.currentText() for col, cb in self.combos.items()},
            "search_text": self.search_box.text(),
            "inc_adv": self.cb_inc_advances.isChecked(),
            "inc_tar": self.cb_inc_tariffs.isChecked(),
            "page_index": self.stacked_widget.currentIndex(),
            "top_n": self.cb_top_n.currentText(),
        }

    def _restore_view_state(self, state: Optional[dict] = None, default_to_settled: bool = False):
        self.updating_filters = True
        selected_filters = state.get("filters", {}) if state else {}
        try:
            for cb in self.combos.values():
                cb.blockSignals(True)
            self.search_box.blockSignals(True)
            self.cb_inc_advances.blockSignals(True)
            self.cb_inc_tariffs.blockSignals(True)
            self.cb_top_n.blockSignals(True)

            for col, cb in self.combos.items():
                cb.clear()
                vals = ["All"] + self.engine.filter_cache.get(col, [])
                cb.addItems(vals)
                target = selected_filters.get(col, "All")
                if target in vals:
                    cb.setCurrentText(target)
                elif col == "SETTLEMENT_STATUS" and default_to_settled:
                    idx = cb.findText("Settled")
                    cb.setCurrentIndex(idx if idx >= 0 else 0)
                else:
                    cb.setCurrentText("All")

            self.search_box.setText(state.get("search_text", "") if state else "")
            self.cb_inc_advances.setChecked(state.get("inc_adv", True) if state else True)
            self.cb_inc_tariffs.setChecked(state.get("inc_tar", True) if state else True)

            top_n_value = state.get("top_n") if state else None
            if top_n_value and self.cb_top_n.findText(top_n_value) >= 0:
                self.cb_top_n.setCurrentText(top_n_value)
        finally:
            for cb in self.combos.values():
                cb.blockSignals(False)
            self.search_box.blockSignals(False)
            self.cb_inc_advances.blockSignals(False)
            self.cb_inc_tariffs.blockSignals(False)
            self.cb_top_n.blockSignals(False)
            self.updating_filters = False

        active_filters = {col: cb.currentText() for col, cb in self.combos.items()}
        self.engine.apply_filters(
            active_filters,
            self.search_box.text().strip(),
            self.cb_inc_advances.isChecked(),
            self.cb_inc_tariffs.isChecked(),
        )
        self.navigate_to_page(state.get("page_index", 0) if state else 0)
        self.update_dashboard()

    def refresh_data_in_background(self):
        self.load_data_from_db(user_initiated=False)

    def load_data_from_db(self, user_initiated: bool = True):
        if self.loader_thread is not None and self.loader_thread.isRunning():
            if user_initiated:
                self.statusBar().showMessage("A data refresh is already running in the background.", 5000)
            return

        self._load_preserve_state = not self.engine.raw_df.empty
        self._load_show_error_dialog = user_initiated or self.engine.raw_df.empty
        self._pending_view_state = self._capture_view_state() if self._load_preserve_state else None
        status_message = "Connecting to FAMOUSODBC and pulling data..." if user_initiated else "Refreshing data from database in the background..."
        self.statusBar().showMessage(status_message)
        self.loader_thread = DataLoaderThread(self.engine)
        self.loader_thread.finished_signal.connect(self.on_data_loaded)
        self.loader_thread.start()

    def on_data_loaded(self, success, msg, df):
        preserved_state = self._pending_view_state
        self._pending_view_state = None
        self.loader_thread = None
        if success and isinstance(df, pd.DataFrame):
            self.engine.apply_loaded_data(df)
            if preserved_state:
                self._restore_view_state(preserved_state)
            else:
                self._restore_view_state(default_to_settled=True)

            current_time = datetime.now().strftime("%Y-%m-%d %I:%M:%S %p")
            self.lbl_timestamp.setText(f"Last Update: {current_time}")
            
            self.statusBar().showMessage(f"Engine Loaded & Calibrated successfully at {current_time}.", 5000)
        else:
            if self._load_show_error_dialog:
                QMessageBox.critical(self, "Database Error", f"Failed to load data from database:\n\n{msg}")
                self.statusBar().clearMessage()
            else:
                self.statusBar().showMessage("Background refresh failed; continuing with cached data.", 5000)

    def reset_filters(self):
        if not self.engine.raw_df.empty: 
            self.search_box.blockSignals(True)
            self.search_box.clear()
            self.search_box.blockSignals(False)
            
            self.cb_inc_advances.blockSignals(True)
            self.cb_inc_advances.setChecked(True)
            self.cb_inc_advances.blockSignals(False)
            
            self.cb_inc_tariffs.blockSignals(True)
            self.cb_inc_tariffs.setChecked(True)
            self.cb_inc_tariffs.blockSignals(False)
            
            self.initialize_filters()

    def initialize_filters(self):
        self.updating_filters = True
        
        for cb in self.combos.values(): cb.blockSignals(True)
        try:
            for col, cb in self.combos.items():
                cb.clear()
                vals = ["All"] + self.engine.filter_cache.get(col, [])
                cb.addItems(vals)
                
            if "SETTLEMENT_STATUS" in self.combos:
                idx = self.combos["SETTLEMENT_STATUS"].findText("Settled")
                if idx >= 0:
                    self.combos["SETTLEMENT_STATUS"].setCurrentIndex(idx)
                else:
                    self.combos["SETTLEMENT_STATUS"].setCurrentText("All")
        finally:
            for cb in self.combos.values(): cb.blockSignals(False)
            self.updating_filters = False

        self.on_filter_changed()

    def on_filter_changed(self):
        if self.updating_filters or self.engine.raw_df.empty: return

        self.updating_filters = True
        QApplication.setOverrideCursor(Qt.CursorShape.WaitCursor)

        try:
            active_filters = {col: cb.currentText() for col, cb in self.combos.items()}
            search_str = self.search_box.text().strip()
            inc_adv = self.cb_inc_advances.isChecked()
            inc_tar = self.cb_inc_tariffs.isChecked()

            def get_valid_options(col_name):
                runtime_df = self.engine.dynamic_full_df if not self.engine.dynamic_full_df.empty else self.engine.raw_df
                if col_name not in runtime_df.columns:
                    return ["All"]
                mask = pd.Series(True, index=runtime_df.index)
                for k, v in active_filters.items():
                    if k != col_name and v != "All" and k in runtime_df.columns:
                        mask &= runtime_df[k].eq(v)
                return ["All"] + self.engine._safe_sorted_unique(runtime_df.loc[mask, col_name])

            for col, cb in self.combos.items():
                cb.blockSignals(True)
                valid_items = get_valid_options(col)
                curr = cb.currentText()
                cb.clear()
                cb.addItems(valid_items)
                if curr in valid_items:
                    cb.setCurrentText(curr)
                else:
                    cb.setCurrentText("All")
                cb.blockSignals(False)

            self.engine.apply_filters(active_filters, search_str, inc_adv, inc_tar)
            self.update_dashboard()

        except Exception as e:
            QMessageBox.critical(self, "Filter Error", f"An error occurred while filtering data:\n{str(e)}\n\n{traceback.format_exc()}")
        finally:
            self.updating_filters = False
            QApplication.restoreOverrideCursor()

    def update_charts(self):
        if self.engine.filtered_df.empty: return
        
        kpis = self.engine.get_kpis()
        self.kpi_ex_sales.value_label.setText(f"${kpis['Gross Sales']:,.2f}")
        self.kpi_ex_exp.value_label.setText(f"${kpis['Total Costs']:,.2f}")
        self.kpi_ex_opex.value_label.setText(f"${kpis['Operating Expenses']:,.2f}")
        self.kpi_ex_adv.value_label.setText(f"${kpis['Total Advances']:,.2f}")
        self.kpi_ex_tar.value_label.setText(f"${kpis['Total Tariffs']:,.2f}")

        self.kpi_ex_net.value_label.setText(f"${kpis['Net Return']:,.2f}")
        self.kpi_ex_cpb.value_label.setText(f"${kpis['Cost / Box']:.2f}")
        self.kpi_ex_rpb.value_label.setText(f"${kpis['Return / Box']:.2f}")
        self.kpi_ex_rpb_notariff.value_label.setText(f"${kpis['Return / Box (No Tariff)']:.2f}")
        
        self.kpi_ex_alr.value_label.setText(f"${kpis['Avg Lot Return']:,.2f}")
        
        self.kpi_ex_comm.value_label.setText(f"${kpis['Commission Rev']:,.2f}")
        self.kpi_ex_comm_pct.value_label.setText(f"{kpis['Commission %']:.1f}%")
        self.kpi_ex_comm_pb.value_label.setText(f"${kpis['Comm / Box']:.2f}")
        
        self.kpi_ex_agr.value_label.setText(f"${kpis['Avg Grower Return']:,.2f}")
        
        self.kpi_ex_qty.value_label.setText(f"{kpis['Total Boxes']:,.0f}")
        self.kpi_ex_fob.value_label.setText(f"${kpis['Sales / FOB']:.2f}")
        self.kpi_ex_tc.value_label.setText(str(kpis["Total Commodities"]))
        self.kpi_ex_tv.value_label.setText(str(kpis["Total Varieties"]))

        limit_text = self.cb_top_n.currentText()
        limit = int(limit_text.replace("Top ", "")) if "Top" in limit_text else None

        comm_df = self.engine.get_profitability_by(["COMMODITY"])
        if limit: comm_df = comm_df.head(limit)
        
        self.chart_commodity.removeAllSeries()
        for ax in self.chart_commodity.axes(): self.chart_commodity.removeAxis(ax)
        self.view_commodity.set_data(comm_df)
        
        if not comm_df.empty:
            series = QBarSeries()
            bar_set = QBarSet("Net Return")
            categories = comm_df["COMMODITY"].astype(str).tolist()
            for value in comm_df["Net_Return"].astype(float).tolist():
                bar_set.append(value)
            series.append(bar_set)
            
            self.chart_commodity.addSeries(series)
            axisX = QBarCategoryAxis()
            axisX.append(categories)
            self.chart_commodity.addAxis(axisX, Qt.AlignmentFlag.AlignBottom)
            series.attachAxis(axisX)

        cost_df = self.engine.get_cost_breakdown(active_only=True)
        if limit: cost_df = cost_df.head(limit)
        
        self.chart_costs.removeAllSeries()
        for ax in self.chart_costs.axes(): self.chart_costs.removeAxis(ax)
        self.view_costs.set_data(cost_df)
        
        if not cost_df.empty:
            if len(cost_df) <= 8:
                pie_series = QPieSeries()
                for charge_name, total_amount in cost_df[["Charge Name", "Total Amount"]].itertuples(index=False, name=None):
                    pie_series.append(str(charge_name), float(total_amount))
                pie_series.setLabelsVisible(True)
                self.chart_costs.addSeries(pie_series)
            else:
                h_series = QHorizontalBarSeries()
                bar_set = QBarSet("Total Amount")
                cost_df_rev = cost_df.sort_values("Total Amount", ascending=True)
                categories = cost_df_rev["Charge Name"].astype(str).tolist()
                for value in cost_df_rev["Total Amount"].astype(float).tolist():
                    bar_set.append(value)
                h_series.append(bar_set)
                
                self.chart_costs.addSeries(h_series)
                axisY = QBarCategoryAxis()
                axisY.append(categories)
                self.chart_costs.addAxis(axisY, Qt.AlignmentFlag.AlignLeft)
                h_series.attachAxis(axisY)
                axisX = QValueAxis()
                self.chart_costs.addAxis(axisX, Qt.AlignmentFlag.AlignBottom)
                h_series.attachAxis(axisX)

    def get_page_dataframe(self, page_name: str) -> pd.DataFrame:
        builders = {
            "Grower Analysis": lambda: self.engine.get_profitability_by(["GROWER_NAME"]),
            "Grower Benchmarking": self.engine.get_grower_benchmarking,
            "Grower Performance Scorecard": self.engine.get_grower_performance_scorecard,
            "Product Analysis": lambda: self.engine.get_profitability_by(["COMMODITY", "VARIETY"]),
            "Product Intelligence": self.engine.get_product_intelligence,
            "Product DNA Analysis": self.engine.get_product_dna_analysis,
            "Variety Performance": self.engine.get_variety_performance,
            "Attribute Analysis": self.engine.get_attribute_analysis,
            "Price Variance Analysis": self.engine.get_price_variance_analysis,
            "Return Variance Explanation": self.engine.get_return_variance_explanation,
            "Market Benchmarking": self.engine.get_market_benchmarks,
            "Outlier Detection": self.engine.get_outlier_analysis,
            "Opportunity Finder": self.engine.get_opportunity_finder,
            "Profitability Drivers": self.engine.get_profitability_drivers,
            "Executive Scorecards": self.engine.get_executive_scorecards,
            "Lot Analysis": lambda: self.engine.get_profitability_by(["LOT_ID", "Region", "SETTLEMENT_STATUS", "GROWER_NAME", "COMMODITY"]),
            "Tag Analysis": self.engine.get_tag_analysis,
            "Cost Analysis": lambda: self.engine.get_cost_breakdown(active_only=True),
        }
        if page_name not in builders:
            return pd.DataFrame()
        cache_key = self._page_cache_key(page_name)
        if cache_key not in self.engine.current_page_cache:
            self.engine.current_page_cache[cache_key] = builders[page_name]()
        return self.engine.current_page_cache[cache_key].copy(deep=False)

    def refresh_current_page_table(self, index: Optional[int] = None):
        page_name = self.page_names_by_index.get(self.stacked_widget.currentIndex() if index is None else index)
        if not page_name or page_name not in self.tables:
            return
        cache_key = self._page_cache_key(page_name)
        table = self.tables[page_name]
        if getattr(table, "_page_cache_key", None) == cache_key:
            return
        self.update_table(table, self.get_page_dataframe(page_name), cache_key)

    def _page_cache_key(self, page_name: str) -> Tuple:
        return (page_name, self.engine.view_cache_token)

    def update_dashboard(self):
        try:
            if self.engine.raw_df.empty: return

            kpis = self.engine.get_kpis()
            self.kpi_qty.value_label.setText(f"{kpis['Total Boxes']:,.0f}")
            self.kpi_sales.value_label.setText(f"${kpis['Gross Sales']:,.2f}")
            self.kpi_net.value_label.setText(f"${kpis['Net Return']:,.2f}")
            
            self.kpi_grower_ret.value_label.setText(f"{kpis['Grower Return %']:.2f}%")
            self.kpi_return.value_label.setText(f"${kpis['Return / Box']:,.2f}")
            self.kpi_return_notariff.value_label.setText(f"${kpis['Return / Box (No Tariff)']:,.2f}")
            
            self.kpi_comm_rev.value_label.setText(f"${kpis['Commission Rev']:,.2f}")
            self.kpi_comm_pct.value_label.setText(f"{kpis['Commission %']:.2f}%")
            self.kpi_comm_pb.value_label.setText(f"${kpis['Comm / Box']:,.2f}")

            self.update_charts()
            self.refresh_current_page_table()

        except Exception as e:
            QMessageBox.critical(self, "Dashboard Error", f"An error occurred while rendering tables:\n{str(e)}\n\n{traceback.format_exc()}")

    def update_table(self, table_widget: QTableView, df: pd.DataFrame, cache_key: Optional[Tuple] = None):
        model = PandasModel(df, parent=table_widget)
        table_widget._custom_model = model  
        proxy = QSortFilterProxyModel(table_widget)
        proxy.setSourceModel(model)
        proxy.setSortRole(Qt.ItemDataRole.UserRole)
        table_widget._custom_proxy = proxy  
        table_widget._page_cache_key = cache_key
        table_widget.setModel(proxy)

    def handle_dialog_drilldown(self, index: QModelIndex, table: QTableView, current_history: str = "Global Filter"):
        model = table.model()
        if model is None: return

        active_page_name = ""
        for name, tbl in self.tables.items():
            if tbl == table:
                active_page_name = name
                break

        col_header = model.headerData(index.column(), Qt.Orientation.Horizontal, Qt.ItemDataRole.DisplayRole)
        val = model.data(index, Qt.ItemDataRole.DisplayRole)
        if not col_header or not val: return
        col_header = str(col_header).strip().upper()
        
        if col_header == "GROWER NAME":
            dialog = GrowerPerformanceDetailDialog(str(val), self.engine, self)
            dialog.exec()
            return
            
        if col_header == "LOT ID":
            if active_page_name == "Price Variance Analysis":
                grower_idx, comm_idx = -1, -1
                for i in range(model.columnCount()):
                    h = model.headerData(i, Qt.Orientation.Horizontal, Qt.ItemDataRole.DisplayRole)
                    if h == "GROWER NAME": grower_idx = i
                    if h == "COMMODITY": comm_idx = i
                g_name = model.data(model.index(index.row(), grower_idx), Qt.ItemDataRole.DisplayRole) if grower_idx > -1 else ""
                c_name = model.data(model.index(index.row(), comm_idx), Qt.ItemDataRole.DisplayRole) if comm_idx > -1 else ""
                dialog = VarianceExplanationDialog(str(val), g_name, c_name, self.engine, self)
                dialog.exec()
            else:
                details = self.engine.get_lot_details(str(val))
                if details:
                    dialog = LotDetailDialog(str(val), details, self)
                    dialog.exec()
            return

        if col_header in ["PALLET TAG ID", "TAG"]:
            df_tag = self.engine.filtered_df[self.engine.filtered_df["PALLET_TAG_ID"] == str(val)]
            dialog = TagDetailDialog(str(val), df_tag, self)
            dialog.exec()
            return

        if col_header == "COMMODITY":
            df_raw = self.engine.filtered_df[self.engine.filtered_df["COMMODITY"] == str(val)]
            cols_to_sum = [c for c in ["QTY_RECEIVED", "TOTAL_REVENUE", "TOTAL_COSTS", "NET_RETURN"] if c in df_raw.columns]
            df_summary = df_raw.groupby("VARIETY", as_index=False, observed=True)[cols_to_sum].sum()
            df_summary["Return_Per_Box"] = (df_summary["NET_RETURN"] / df_summary["QTY_RECEIVED"].replace(0, np.nan)).fillna(0)
            df_summary = df_summary.sort_values(by="Return_Per_Box", ascending=False)
            dialog = GenericAnalysisWindow(f"Commodity Analysis: {val}", df_summary, df_raw, self)
            dialog.exec()
            return

        if col_header == "VARIETY":
            df_raw = self.engine.filtered_df[self.engine.filtered_df["VARIETY"] == str(val)]
            cols_to_sum = [c for c in ["QTY_RECEIVED", "TOTAL_REVENUE", "TOTAL_COSTS", "NET_RETURN"] if c in df_raw.columns]
            df_summary = df_raw.groupby(["GRADE", "SIZENAME"], as_index=False, observed=True)[cols_to_sum].sum()
            df_summary["Return_Per_Box"] = (df_summary["NET_RETURN"] / df_summary["QTY_RECEIVED"].replace(0, np.nan)).fillna(0)
            df_summary = df_summary.sort_values(by="Return_Per_Box", ascending=False)
            dialog = GenericAnalysisWindow(f"Variety Analysis: {val}", df_summary, df_raw, self)
            dialog.exec()
            return

    def export_table_to_excel(self, page_name: str):
        if page_name not in self.tables: return
        table = self.tables[page_name]
        proxy = table.model()
        if not proxy: return
        source = proxy.sourceModel()
        if not isinstance(source, PandasModel): return
        df = source.get_dataframe()
        export_df_to_excel(df, page_name, self)

if __name__ == "__main__":
    app = QApplication(sys.argv)
    app.setStyle("Fusion")
    window = MainWindow()
    window.show()
    sys.exit(app.exec())
