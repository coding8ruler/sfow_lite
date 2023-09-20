# sfow_lite / 스마트팩토리
TOAST GRID_웹기반 클라우드 보급형 LITE형 ERP MES 개발

## 🖥️ 프로젝트 소개
(주)여누솔루션 시스템을 참고하여 만든 스마트팩토리입니다.
<br>

## 🕰️ 개발 기간 & 참여인원
* 22.03.14일 - 22.04.13일
* 15명

### 🧑‍🤝‍🧑 역할
 - 품목별 재고 기능 및 페이지 구현
 - ERDcloud를 활용한테이블 구조 (상관관계) 정리 DB 설계

### ⚙️ 개발 환경
- `Java 8`
- `JDK 1.8.0`
- `Javascript`
- `HTML5`
- `CSS3`
- **IDE** : eclipse 2020-03
- **Framework** :Spring(3.9.12.RELEASE)
- **Libraries** :jQuery, AJAX, TOAST UI Grid
- **Database** : MariaDB 10.6.7
- **ORM** : Mybatis
- **ETC** : git, notion, Apache Tomcat

## ✨ 핵심 기능 - 자동화된 재고 관리를 위한 SQL 트리거
```
-- 구매 update (재고 수량 증가)
DROP TRIGGER IF EXISTS tr_update_mt_stock_detail_po_in_detail;

DELIMITER $$
CREATE TRIGGER tr_update_mt_stock_detail_po_in_detail
AFTER UPDATE ON po_in_detail
FOR EACH ROW
BEGIN
    IF (NEW.in_order IS NOT NULL) THEN
        UPDATE mt_stock_detail SET quantity = quantity + (NEW.in_quantity - IFNULL(OLD.in_quantity, 0)) WHERE mt_stock_detail.item_code = NEW.item_code;
    ELSEIF (OLD.in_order IS NOT NULL) THEN
        UPDATE mt_stock_detail SET quantity = quantity - OLD.in_quantity WHERE mt_stock_detail.item_code = OLD.item_code;
    END IF;
END$$

-- 구매 insert
DROP TRIGGER IF EXISTS tr_INSERT_mt_stock_detail_po_in_detail;

DELIMITER $$
CREATE TRIGGER tr_INSERT_mt_stock_detail_po_in_detail
AFTER INSERT ON po_in_detail
FOR EACH ROW
BEGIN
    DECLARE companyCode VARCHAR(10);
    SELECT company_code INTO companyCode FROM po_request WHERE request_number = (SELECT request_number FROM po_in WHERE in_number = NEW.in_number) LIMIT 1;
    IF (NEW.in_order IS NOT NULL) THEN
        INSERT INTO mt_stock_detail (company_code, item_code, quantity, warehouse_code) VALUES (companyCode, NEW.item_code, NEW.in_quantity, NEW.warehouse_code) 
        ON DUPLICATE KEY UPDATE quantity = quantity + NEW.in_quantity;
    END IF;
END$$
DELIMITER ;


-- 구매 delete
DROP TRIGGER IF EXISTS tr_DELETE_mt_stock_detail_po_in_detail;

DELIMITER $$
CREATE TRIGGER tr_DELETE_mt_stock_detail_po_in_detail
AFTER DELETE ON po_in_detail
FOR EACH ROW
BEGIN
    IF (OLD.in_order IS NOT NULL) THEN
        UPDATE mt_stock_detail
        SET quantity = quantity - OLD.in_quantity
        WHERE item_code = OLD.item_code;
    END IF;
END$$

-- 구매 Y,N DELETE
DROP TRIGGER IF EXISTS tr_updateDELETE_mt_stock_detail_po_in_detail;
DELIMITER $$
CREATE TRIGGER tr_updateDELETE_mt_stock_detail_po_in_detail
AFTER UPDATE ON po_in
FOR EACH ROW
BEGIN
    IF (NEW.in_delyn = 'Y') THEN
        UPDATE mt_stock_detail INNER JOIN po_in_detail ON mt_stock_detail.item_code = po_in_detail.item_code 
        SET mt_stock_detail.quantity = mt_stock_detail.quantity - po_in_detail.in_quantity 
        WHERE po_in_detail.in_number = NEW.in_number;
    END IF;
END$$

COMMIT;
-- --------------------------------------------------------------------------------------------------
-- --------------------------------------------------------------------------------------------------

-- -----------------------------------------------
-- 출고 딜리트(수량 증가)와 업데이트 , 인서트는 등록으로 등록돼서 할필요 없음, 그리고 출고니까 재고에 있는 제품만 되니까 아이템코드 등록될 필요 없음
DROP TRIGGER IF EXISTS tr_update_mt_stock_detail_so_shipout;

DELIMITER $$
CREATE TRIGGER tr_update_mt_stock_detail_so_shipout
AFTER UPDATE ON so_shipout
FOR EACH ROW
BEGIN
    DECLARE item_code_list TEXT;
    DECLARE qty_list TEXT;

    IF (OLD.out_status = '등록' AND NEW.out_status = '확정' AND NEW.delete_yes_no <> 'Y') THEN
        SELECT GROUP_CONCAT(item_code), GROUP_CONCAT(quantity) INTO item_code_list, qty_list 
        FROM so_order_detail 
        WHERE order_number = NEW.order_number;
        
        shipout_loop: WHILE (LENGTH(item_code_list) > 0) DO
            SET @item_code = SUBSTRING_INDEX(item_code_list, ',', 1);
            SET @qty = SUBSTRING_INDEX(qty_list, ',', 1);
            
            IF (@qty <= (SELECT quantity FROM mt_stock_detail WHERE item_code = @item_code)) THEN
                UPDATE mt_stock_detail SET quantity = quantity - @qty WHERE item_code = @item_code;
                SET item_code_list = SUBSTRING(item_code_list, LENGTH(@item_code) + 2);
                SET qty_list = SUBSTRING(qty_list, LENGTH(@qty) + 2);
            ELSE
                SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '재고 수량 부족';
                LEAVE shipout_loop;
            END IF;
        END WHILE shipout_loop;

    ELSEIF (OLD.out_status = '확정' AND OLD.delete_yes_no = 'N' AND NEW.delete_yes_no = 'Y') THEN
        SELECT GROUP_CONCAT(item_code), GROUP_CONCAT(quantity) INTO item_code_list, qty_list 
        FROM so_order_detail 
        WHERE order_number = NEW.order_number;

        refund_loop: WHILE (LENGTH(item_code_list) > 0) DO
            SET @item_code = SUBSTRING_INDEX(item_code_list, ',', 1);
            SET @qty = SUBSTRING_INDEX(qty_list, ',', 1);

            UPDATE mt_stock_detail SET quantity = quantity +  @qty WHERE item_code = @item_code;
            SET item_code_list = SUBSTRING(item_code_list, LENGTH(@item_code) + 2);
            SET qty_list = SUBSTRING(qty_list, LENGTH(@qty) + 2);
        END WHILE refund_loop;
        
    ELSEIF (OLD.out_status = '확정' AND NEW.out_status = '등록' AND NEW.delete_yes_no <> 'Y') THEN
        SELECT GROUP_CONCAT(item_code), GROUP_CONCAT(quantity) INTO item_code_list, qty_list 
        FROM so_order_detail 
        WHERE order_number = NEW.order_number;

        register_loop: WHILE (LENGTH(item_code_list) > 0) DO
            SET @item_code = SUBSTRING_INDEX(item_code_list, ',', 1);
            SET @qty = SUBSTRING_INDEX(qty_list, ',', 1);

            UPDATE mt_stock_detail SET quantity = quantity +  @qty WHERE item_code = @item_code;
            SET item_code_list = SUBSTRING(item_code_list, LENGTH(@item_code) + 2);
            SET qty_list = SUBSTRING(qty_list, LENGTH(@qty) + 2);
        END WHILE register_loop;
    END IF;
END $$
DELIMITER ;

-- -------------------------------------------------------------------------------
-- -------------------------------------------------------------------------
DROP TRIGGER IF EXISTS tr_update_mt_stock_detail_so_return_add;
-- 반품 업데이트add
DELIMITER $$
CREATE TRIGGER tr_update_mt_stock_detail_so_return_add
AFTER UPDATE ON so_return_add
FOR EACH ROW
BEGIN
DECLARE return_item_code VARCHAR(20);
    IF (NEW.out_status = 1) THEN
        SET return_item_code = (SELECT item_code FROM so_return_detail WHERE return_number = NEW.return_number);
        IF (NOT EXISTS (SELECT * FROM mt_stock_detail WHERE item_code = return_item_code)) THEN
            INSERT INTO mt_stock_detail (item_code, quantity) VALUES (return_item_code, (SELECT item_quantity FROM so_return_detail WHERE return_number = NEW.return_number AND item_code = mt_stock_detail.item_code));
        ELSE
            UPDATE mt_stock_detail
            SET quantity = quantity + (SELECT item_quantity FROM so_return_detail WHERE return_number = NEW.return_number AND item_code = mt_stock_detail.item_code)
            WHERE mt_stock_detail.item_code = return_item_code;
        END IF;
    END IF;SFOW_LITE
END$$

-- 반품 업데이트 detail
DROP TRIGGER IF EXISTS tr_update_mt_stock_detail_so_return_detail;

DELIMITER $$
CREATE TRIGGER tr_update_mt_stock_detail_so_return_detail
AFTER UPDATE ON so_return_detail
FOR EACH ROW
BEGIN
  IF (SELECT out_status FROM so_return_add WHERE return_number = NEW.return_number) = 1 THEN
    UPDATE mt_stock_detail
      SET quantity = quantity + NEW.item_quantity
      WHERE item_code = NEW.item_code;
  END IF;
END$$

-- 반품 삭제 add
DROP TRIGGER IF EXISTS tr_delete_mt_stock_detail_so_return_add;
DELIMITER $$
CREATE TRIGGER tr_delete_mt_stock_detail_so_return_add
AFTER UPDATE ON so_return_add
FOR EACH ROW
BEGIN
  IF (OLD.out_status = 1 AND NEW.out_status = 0) THEN
    UPDATE mt_stock_detail
      SET quantity = quantity - (SELECT item_quantity FROM so_return_detail WHERE return_number = NEW.return_number AND item_code = mt_stock_detail.item_code)
      WHERE mt_stock_detail.item_code IN (SELECT item_code FROM so_return_detail WHERE return_number = NEW.return_number);
  END IF;
END$$
DELIMITER ;

-- 반품 삭제 detail
DROP TRIGGER IF EXISTS tr_delete_mt_stock_detail_so_return_detail;

DELIMITER $$
CREATE TRIGGER tr_delete_mt_stock_detail_so_return_detail
AFTER DELETE ON so_return_detail
FOR EACH ROW
BEGIN
  IF (SELECT out_status FROM so_return_add WHERE return_number = OLD.return_number) = 1 THEN
    UPDATE mt_stock_detail
      SET quantity = quantity - OLD.item_quantity
      WHERE item_code = OLD.item_code;
  END IF;
END$$
DELIMITER ;


COMMIT;

-- ---------------------------------------------------------------------------
-- 생산 인서트
-- 생산 업데이트
DROP TRIGGER IF EXISTS tr_update_mt_stock_detail_pp_perform;
-- 이건 no 사용 sitem_cd 안됨
DELIMITER $$
CREATE TRIGGER tr_update_mt_stock_detail_pp_perform
AFTER UPDATE ON pp_perform
FOR EACH ROW
BEGIN
DECLARE ppitem_cd VARCHAR(100);
DECLARE sitem_cd VARCHAR(100);
DECLARE item_qty INT(11);

IF (NEW.per_status = 'E') THEN
    SELECT ppitem_cd, sitem_cd, item_qty 
    INTO ppitem_cd, sitem_cd, item_qty 
    FROM ma_bom_tree 
    WHERE bomdno = NEW.bomdno;

    UPDATE mt_stock_detail 
    SET quantity = quantity + NEW.per_quantity 
    WHERE item_code = ppitem_cd;

    UPDATE mt_stock_detail 
    SET quantity = quantity - item_qty 
    WHERE item_code = sitem_cd;
END IF;
END$$
-- no 사용
-- pp_perform e되면 재고에서 sitem 빼는 
DELIMITER $$
CREATE TRIGGER tr_update_mt_stock_detail_pp_perform
AFTER UPDATE ON pp_perform
FOR EACH ROW
BEGIN
DECLARE ssitem_cd VARCHAR(100);
DECLARE iitem_qty INT(11);

IF (NEW.per_status = 'E') THEN
    SELECT sitem_cd, item_qty 
    INTO ssitem_cd, iitem_qty 
    FROM ma_bom_tree 
    WHERE bomdno = NEW.bomdno;

    UPDATE mt_stock_detail 
    SET quantity = quantity - iitem_qty 
    WHERE item_code = ssitem_cd;
END IF;
END$$
DELIMITER ;
-- ma_bom_tree에서 sitem 수량 변경되면 변경하는


```
#### 품목별재고현황 그리드 - <a href="https://github.com/coding8ruler/sfow_lite/blob/master/sfow_lite/src/main/webapp/WEB-INF/views/stock/stockByItemMain.jsp" >상세보기 - 이동</a>

## 📌 
- 그리드 기능 구현
- 스마트팩토리 이해
- 15인 프로젝트 규모에서의 커뮤니케이션의 중요성 이해
- TOAST GRID 활용 및 자체 문법 학습
- SMART FACTORY 생산 구조 파악
- 실제 상용 솔루션에서의 사용자 중심 개발의 중요성이해
- AJAX필요성과대용량 데이터를 보다 효율적으로 표현, 관리하기 위한 데이터 구조의 
이해

