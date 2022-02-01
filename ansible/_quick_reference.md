# AD-HOC Exec

Forma unitária e simplificada. Antendar que a vírugula seguinte ao nome do host 
é essencial se utilizado um hostname ao invés de informar um arquivo de inventário.

```
ansible -i host1, -u zeca -k -m shell -a hostname all
```
