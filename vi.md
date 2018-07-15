[Nguồn](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Permalink to Compound (Composite) Indexes - MariaDB Knowledge Base")

- Translated by Tien Nguyen

# Chỉ mục hỗn hợp (Composite Index) - Kiến thức về MariaDB cơ bản 

## Một bài học nhỏ trong "Chỉ mục hỗn hợp" ("composite Index")

Tài liệu này bắt đầu có vẻ tầm thường và nhàm chán, nhưng xây dựng lên nhiều thông tin thú vị hơn, có lẽ bạn không nhận ra về cách mà chỉ mục (index) của MariaDB và MySQL hoạt động.

Nó cũng giải thích [EXPLAIN][1] ( đến một mức độ nào đó).

( Hầu hết điều này cũng áp dụng cho các loại cơ sở dữ liệu không phải MySQL)

## Truy vấn để thảo luận

Câu hỏi là "Andrew Johnson đã trở thành tổng thống Mỹ khi nào?".

Bảng `Presidents` có sẵn trông giống như:
    
    
    +-----+------------+----------------+-----------+
    | seq | last_name  | first_name     | term      |
    +-----+------------+----------------+-----------+
    |   1 | Washington | George         | 1789-1797 |
    |   2 | Adams      | John           | 1797-1801 |
    ...
    |   7 | Jackson    | Andrew         | 1829-1837 |
    ...
    |  17 | Johnson    | Andrew         | 1865-1869 |
    ...
    |  36 | Johnson    | Lyndon B.      | 1963-1969 |
    ...
    

("Andrew Johnson" đã được chọn cho bài học này bởi vì những sự trùng lặp)

Chỉ mục(nhiều chỉ mục) nào là tốt nhất cho câu hỏi đó? Cụ thể hơn, cái nào là tốt nhất cho
    
    
        SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
    

Một vài INDEX để thử...

* Không chỉ mục
* INDEX(first_name), INDEX(last_name) (hai chỉ mục riêng biệt) 
* "Index Merge Intersect"
* INDEX(last_name, first_name) (một chỉ mục "compound") 
* INDEX(last_name, first_name, term) (một chỉ mục bao hàm) 
* Biến thể 

## Không chỉ mục

Tốt thôi, Tôi đang vớ vẩn một chút ở đây. Tôi có một KHÓA CHÍNH (PRIMARY KEY) tại `seq`, nhưng nó không có lợi ích trong truy vấn mà chúng ta đang học.
    
    
    mysql>  SHOW CREATE TABLE Presidents G
    CREATE TABLE `presidents` (
      `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
      `last_name` varchar(30) NOT NULL,
      `first_name` varchar(30) NOT NULL,
      `term` varchar(9) NOT NULL,
      PRIMARY KEY (`seq`)
    ) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew';
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    | id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    |  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    
    # Or, using the other form of display:  EXPLAIN ... G
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ALL        <-- Ngụ ý là scan bảng
    possible_keys: NULL
              key: NULL       <-- Ngụ ý là không có index nào hữu ích, do đó scan bảng
          key_len: NULL
              ref: NULL
             rows: 44         <-- Điều này là có bao nhiêu hàng trong bảng, vậy scan bảng
            Extra: Using where
    

## Chi tiết triển khai

Đầu tiên, hãy giới thiệu cách InnoDB lưu trữ và sử dụng chỉ mục.

* Dữ liệu và KHÓA CHÍNH (PRIMARY KEY) nhóm lại cùng nhau trên BTree.
* Tra cứu BTree khá nhanh và hiệu quả. Cho một bảng với hàng triệu hàng có lẽ có 3 cấp độ của BTree, và hai level cao nhất được lưu vào cache.
* Mỗi chỉ mục thứ cấp trong một BTree khác, với KHÓA CHÍNH ở lá.
* Việt lấy liên tục ( theo chỉ mục ) các phần tử từ một BTree là vô cùng hiệu quả bởi vì chúng được lưu trữ liên tục.
* Với lợi ích đơn giản, chúng ta có thể đếm mỗi lần tra cứu BTree như 1 đơn vị công việc, và loại bỏ scan phần tử liên tục. Nó xấp xỉ con số truy cập của ổ đĩa cho một bảng lớn trong một hệ thống bận.

Với MyISAM, KHÓA CHÍNH không được lưu trữ với dữ liệu, vậy suy nghĩ nó giống như một khóa thứ cấp ( quá đơn giản ).

## INDEX(first_name), INDEX(last_name)

Người mới, mỗi lần anh ấy học về việc đánh chỉ mục, quyết định để lập chỉ mục của nhiều cột, một cái một lần. Nhưng...

MySQL hiếm khi sử dụng nhiều hơn một chỉ mục trong một lần trong một truy vấn. Vậy nó sẽ phân tích những chỉ mục có thể.

* first_name -- có hai hàng có thể (một tra cứu BTree, sau đó scan liên tục)
* last_name -- có hai hàng có thể. Giả sử nó chọn last_name. Đây là những bước cho việc SELECT:
1. Sử dụng INDEX(last_name), tìm 2 chỉ mục với last_name = 'Johnson'.
2. Lấy KHÓA CHÍNH (đã ngầm thêm vào mỗi chỉ mục thứ cấp trong )InnoDB; lấy (17, 36). 
3. Tiếp cận dữ liệu sử dụng seq = (17, 36) để lấy những hàng cho Andrew Johnson và Lyndon B. Johnson. 
4. Sử dụng phần còn lại của mệnh đề WHERE lọc tất cả những trừ hàng mong muốn.
5. Cung cấp câu trả lời (1865-1869). 

```
mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew'  G
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: last_name, first_name
              key: last_name
          key_len: 92                 <-- VARCHAR(30) utf8 cần 2+3*30 bytes
              ref: const
             rows: 2                  <-- Hai 'Johnson's
            Extra: Using where
```


## "Index Merge Intersect" 

OK, vậy bạn trở thành cực kỳ thông minh và quyết định rằng MySQL nên đủ thông minh để sử dụng tên chỉ mục giống nhau để có câu trả lời. Điều này được gọi là "Intersect".
1. Sử dụng INDEX(last_name), tìm 2 chỉ mục với last_name = 'Johnson'; nhận được (7, 17) 
2. Sử dụng INDEX(first_name), tìm 2 chỉ mục với first_name = 'Andrew'; nhận được (17, 36) 
3. "And" hai danh sách cùng nhau (7,17) & (17,36) = (17) 
4. Tiếp cận dữ liệu sử dụng seq = (17) để có được hàng cho Andrew Johnson. 
5. Cung cấp câu trả lời (1865-1869). 
    
``` id: 1
      select_type: SIMPLE
            table: Presidents
             type: index_merge
    possible_keys: first_name,last_name
              key: first_name,last_name
          key_len: 92,92
              ref: NULL
             rows: 1
            Extra: Using intersect(first_name,last_name); Using where 
```


Câu lệnh EXPLAIN lỗi để cho ra thông tin chi tiết của bao nhiêu hàng được thu thập từ mỗi chỉ mục, vân vân.

## INDEX(last_name, first_name)

Đó họ là "compound" hoặc "composite" index khi nó có nhiều hơn một cột.
1. Đi sâu vào BTree để đánh chỉ mục để có được chính xác chỉ mục của hàng cho Johnson+Andrew; có được seq = (17). 
2. Tiếp cận dữ liệu sử dụng seq = (17) để có được hàng cho Andrew Johnson. 
3. Cung cấp câu trả lời (1865-1869). Nó tốt hơn nhiều. Trong thực tế nó được gọi là "best".


``` 
ALTER TABLE Presidents
            (drop old indexes and...)
            ADD INDEX compound(last_name, first_name);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: compound
              key: compound
          key_len: 184             <-- Độ dài của cả 2 trường
              ref: const,const     <-- Mệnh đề WHERE trả về hằng cho cả 2
             rows: 1               <-- Goodie!  It homed in on the one row.
            Extra: Using where
```


## "Bao hàm": INDEX(last_name, first_name, term)

Bất ngờ chưa! Chúng ta thực ra có thể làm tốt hơn một chút. Một chỉ mục "bao hàm" là một trong cái _all_ của các trường của SELECT được tìm thấy trong chỉ mục. Nó có điểm cộng thêm là không phải tiếp cận vào "dữ liệu" để hoàn thành nhiệm vụ.
1. Đi sâu vào BTree để đánh chỉ mục để có được chính xác chỉ mục của hàng cho Johnson+Andrew; có được seq = (17). 
2. Cung cấp câu trả lời (1865-1869). Dữ liệu BTree chưa được chạm vào; điều này là sự cải tiến hơn "composite".
    
```    
        ... ADD INDEX covering(last_name, first_name, term);
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: covering
              key: covering
          key_len: 184
              ref: const,const
             rows: 1
            Extra: Using where; Using index   <-- Note 
```

Mọi thứ tương tự để sử dụng "compound", ngoại trừ việc bổ sung "sử dụng chỉ mục".

## Biến thể

* Chuyện gì xảy ra nếu bạn xáo trộn những trường trong mệnh đề WHERE? Câu trả lời: Thứ tự của AND không quan trọng.
* Chuyện gì xảy ra nếu bạn xáo trộn những trường trong mệnh đề INDEX? Câu trả lời: Nó có lẽ tại ta một sự khác biệt lớn. Nhiều hơn trong một phút.
* Chuyện gì nếu có những trường thêm ở cuối? Câu trả lời: Tác hại tối thiểu; có thể có nhiều cái hay (ví dụ: 'bao hàm').
* Thừa thãi ư? Đúng vậy, chuyện gì nếu bạn có cả 2 thứ này: INDEX(a), INDEX(a,b)? Câu trả lời: thừa chi phí gì đó trên câu lệnh INSERT; Nó hiếm khi dùng cho câu lệnh SELECT.
* Tiền tố? Đúng vậy, INDEX(last_name(5). first_name(5)). Câu trả lời: Đừng bận tâm; nó hiếm khi giúp và thường có hại. (Những chi tiết là một chủ đề khác).

## Ví dụ khác:
    
    
        INDEX(last, first)
        ... WHERE last = '...' -- good (even though `first` is unused)
        ... WHERE first = '...' -- index is useless
    
        INDEX(first, last), INDEX(last, first)
        ... WHERE first = '...' -- 1st index is used
        ... WHERE last = '...' -- 2nd index is used
        ... WHERE first = '...' AND last = '...' -- either could be used equally well
    
        INDEX(last, first)
        Both of these are handled by that one INDEX:
        ... WHERE last = '...'
        ... WHERE last = '...' AND first = '...'
    
        INDEX(last), INDEX(last, first)
        In light of the above example, don't bother including INDEX(last).
    

## Postlog

Refreshed -- Oct, 2012; more links -- Nov 2016
