## Mybatis高效率批量更新

#### 常规方式

- 循环执行update语句，mybatis动态sql如下：

  ```
  <update id="batchUpdate"  parameterType="java.util.List">  
      <foreach collection="list" item="item" index="index" open="" close="" separator=";">
          UPDATE table_1
          <set>
              field_1 = ${item.field1}
          </set>
          <where>
          		field_2 = ${item.field2}
          </where>
      </foreach>      
  </update>
  ```

- 本质上还是循环执行单条update语句，当批量更新大量记录时性能较差。

#### case-when方式

- MySQL没有提供直接的方法来实现批量更新，但可以使用case-when语法来实现这个功能。

  ```
  UPDATE table_1
      SET name = CASE  
          WHEN id = 1 THEN 'name1'
          WHEN id = 2 THEN 'name2'
          WHEN id = 3 THEN 'name3'
      END, 
      title = CASE 
          WHEN id = 1 THEN 'New Title 1'
          WHEN id = 2 THEN 'New Title 2'
          WHEN id = 3 THEN 'New Title 3'
      END
  WHERE id IN (1,2,3)
  ```

- 单字段查询条件的mybatis动态sql如下：

  ```
  <update id="batchUpdate" parameterType="java.util.List">
      UPDATE table_1
    		SET name = CASE
          <foreach collection="list" item="item" index="index" separator="">
            WHEN id = #{item.id} THEN #{item.name} 
          </foreach>
    		END,
    		title = CASE
          <foreach collection="list" item="item" index="index" separator="">
          	WHEN id = #{item.id} THEN #{item.title} 
          </foreach>
    		END
    	WHERE id in
      <foreach collection="list" item="item" index="index" open="(" separator="," close=")">
        #{item.id} 
      </foreach>
  </update>
  ```

- 多字段查询条件的mybatis动态sql如下：

  ```
  <update id="batchUpdate" parameterType="java.util.List">
      UPDATE table_1
    		SET name = CASE
          <foreach collection="list" item="item" index="index" separator="">
            WHEN id_1 = #{item.id1} and id_2 = #{item.id2} THEN #{item.name} 
          </foreach>
    		END,
    		title = CASE
          <foreach collection="list" item="item" index="index" separator="">
            WHEN id_1 = #{item.id1} and id_2 = #{item.id2} THEN #{item.title} 
          </foreach>
    		END
    	WHERE (id_1, id_2) in
      <foreach collection="list" item="item" index="index" open="(" separator="," close=")">
        (#{item.id1}, #{item.id2}) 
      </foreach>
  </update>
  ```

  