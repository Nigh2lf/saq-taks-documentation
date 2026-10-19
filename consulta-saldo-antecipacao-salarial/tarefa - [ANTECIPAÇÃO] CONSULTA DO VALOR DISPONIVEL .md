# Cálculo e consulta do saldo disponível para antecipação salarial

## Objetivo

Implementar a consulta do saldo disponível para antecipação salarial do usuário autenticado.

O sistema deverá calcular o limite mensal do colaborador com base em **30% do salário bruto para pf** e descontar desse limite os valores que já foram antecipados no período.
Para PJ a porcentagem deve variar

---

## Regra de cálculo

O limite mensal para antecipação deverá corresponder a 30% do salário bruto do colaborador.

```text
Limite mensal = Salário bruto × 30%
```

Em seguida, o sistema deverá consultar a tabela de operações e somar todas as antecipações realizadas pelo colaborador no mês de referência.

```text
Saldo disponível = Limite mensal − Total antecipado no mês
```

### Exemplo

Considerando um colaborador com salário bruto de R$ 5.000,00:

```text
Limite mensal = R$ 5.000,00 × 30%
Limite mensal = R$ 1.500,00
```

Caso o colaborador já tenha antecipado R$ 600,00 no mês:

```text
Saldo disponível = R$ 1500,00 − R$ 600,00
Saldo disponível = R$ 900,00
```

---

## Fluxo

1. Identificar o colaborador relacionado ao usuário autenticado;
2. Consultar o salário bruto cadastrado;
3. Calcular 30% do salário bruto;
4. Consultar as operações de antecipação do colaborador no mês de referência;
5. Somar os valores já antecipados no período;
6. Deduzir o total antecipado do limite mensal;
7. Retornar o limite mensal, o total antecipado e o saldo ainda disponível.

---

## Data de vencimento

A data de vencimento da operação deverá ser considerada para identificar corretamente o mês de referência utilizado no cálculo.

Somente as operações pertencentes ao mesmo período de vencimento deverão ser descontadas do limite disponível daquele mês.

Ao iniciar um novo período de vencimento, o limite deverá ser recalculado, sem descontar operações referentes aos períodos anteriores.

---

## Regras de negócio

* O cálculo deverá ser realizado somente para o usuário autenticado;
* O limite mensal deverá corresponder a 30% do salário bruto;
* Todas as operações do mês de referência deverão ser consideradas no cálculo;
* Operações canceladas, recusadas ou que não tenham gerado liberação financeira não deverão consumir o limite;
* O saldo disponível deverá ser atualizado após cada nova antecipação;
* O saldo disponível nunca poderá ser menor que zero;
* O usuário não poderá solicitar um valor superior ao saldo disponível;
* Caso o salário bruto não esteja cadastrado, o cálculo não deverá ser realizado;
* A data de vencimento deverá definir o período utilizado para consulta das operações.

---

## Critérios de aceite

* [ ] O saldo é calculado somente para o usuário autenticado;
* [ ] O limite mensal corresponde a 30% do salário bruto cadastrado;
* [ ] O total antecipado considera apenas as operações do período de vencimento de referência;
* [ ] Operações canceladas, recusadas ou sem liberação financeira não consomem o limite;
* [ ] O saldo disponível é atualizado após cada nova antecipação;
* [ ] O saldo disponível nunca retorna valor negativo;
* [ ] Solicitações acima do saldo disponível são rejeitadas;
* [ ] Quando não há salário bruto cadastrado, o cálculo não é realizado e a API retorna erro adequado;
* [ ] A resposta retorna limite mensal, total antecipado e saldo disponível;
* [ ] Testes cobrindo o cálculo, a virada de período e as regras de exclusão de operações.

SELECT *  from payroll_employee 

SELECT * from payroll_employer pe 

SELECT * from contract_creditagreement cc 

SELECT * from contract_loanposition 