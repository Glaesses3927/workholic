<script setup>
  import { ref, onMounted } from 'vue'
  import Modal from './Modal.vue';
  const showInfoModal = ref(false);
  const showDetailModal = ref(false);
  const showCautionModal = ref(false);
  const nwh = ref(8)
  const shiftsText = ref("")
  const result = ref([])
  const STORAGE_KEY = 'workholic_session_entries'
  const entries = ref([])

  const timeRangePattern = /^(?:[0-1]?\d|2[0-9]):[0-5]\d-(?:\(\+\))?(?:[0-1]?\d|2[0-9]):[0-5]\d/;

  function parseShiftData() {
    const lines = shiftsText.value.split('\n').map(line => line.trim()).filter(line => line !== '');
    const shifts = [];
    let currentDay = null;
    let currentShift = null;
    let days = 0;
    const p = lines[0].match(/(1?[0-9])\/([1-3]?[0-9])/);
    let period = p[1] + '月';
    if(1<=parseInt(p[2]) && parseInt(p[2])<=15) period += '前半';
    else if(16<=parseInt(p[2]) && parseInt(p[2])<=31) period += '後半';
    else throw new Error('[ERROR] Code:ERR01');

    lines.forEach(line => {
      const dayMatch = line.match(/(1?[0-9]\/[1-3]?[0-9])\((Su|Mo|Tu|We|Th|Fr|Sa)\)/);
      if (dayMatch) {
        days++;
        currentDay = dayMatch;
      } else if (line.includes('休暇')) {
        shifts.push({ day: currentDay[1], dow: currentDay[2], begin: '', end: '', rest: '', name: '休暇', wh: nwh.value+'h' });
      } else if (line == "----" || line == "休日"){
        days--;
        return;
      } else if (timeRangePattern.test(line)) {
        const [timePart, whPart] = line.trim().split(/\s+/);
        const [begin, end] = timePart.split('-');
        currentShift = {
          day: currentDay[1],
          dow: currentDay[2],
          begin,
          end,
          rest: '0h',
          name: '',
          wh: (whPart || '').replace(/[()]/g, '')
        };
      } else if (line.startsWith('[休')) {
        const restMatch = line.match(/\((\d+h\d*)\)/);
        if (restMatch) {
          currentShift.rest = restMatch[1];
        }
      } else if(line.includes('欠勤')) {
        days--;
        currentShift.name = line;
        shifts.push(currentShift);
      } else {
        currentShift.name = line;
        shifts.push(currentShift);
      }
    });
    return [shifts, period, days];
  }
  function calculateTotalWorkHours(shifts) {
    let totalMinutes = 0;
    shifts.forEach(shift => {
        let workHours = 0;
        if (shift.wh && !shift.name.includes("欠勤")) {
            const [hours, minutes] = shift.wh.split('h').map(str=>parseInt(str, 10));
            workHours = (hours||0)*60 + (minutes||0);
        }
        totalMinutes += workHours;
    });
    const totalHours = Math.floor(totalMinutes / 60);
    const remainingMinutes = totalMinutes % 60;
    return `${totalHours}h${remainingMinutes > 0 ? remainingMinutes + 'm' : ''}`;
  }
  function calc(){
    try {
      const originalShifts = shiftsText.value;
      const [shifts, period, days] = parseShiftData();
      const hours = calculateTotalWorkHours(shifts);
      const res = { period, hours, days };
      result.value.push(res);
      entries.value.push({ shiftsText: originalShifts, nwh: nwh.value, result: res });
      localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value));
      shiftsText.value = "";
    } catch (error) {
      alert(error);
    }
  }
  function dele(i){
    result.value.splice(i,1);
    entries.value.splice(i,1);
    localStorage.setItem(STORAGE_KEY, JSON.stringify(entries.value));
  }
  function setShowInfoModal(){
    showInfoModal.value = !showInfoModal.value
  }
  function setShowDetailModal(){
    showDetailModal.value = !showDetailModal.value
  }
  function setShowCautionModal(){
    showCautionModal.value = !showCautionModal.value
  }

  onMounted(() => {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (!raw) return;
      const saved = JSON.parse(raw);
      if (!Array.isArray(saved)) return;
      entries.value = saved;
      result.value = saved.map(e => e.result).filter(Boolean);
      if (saved.length > 0) {
        const last = saved[saved.length - 1];
        if (last && typeof last.nwh !== 'undefined') nwh.value = last.nwh;
      }
    } catch (e) {
      console.warn('Failed to load session entries', e);
    }
  })
</script>


<template>
  <div class="p-8 sm:p-24 w-full max-w-[700px]">
    <form @submit.prevent="calc">
      <div class="w-full mb-4 border border-gray-200 rounded-lg bg-gray-50 shadow-md">
        <div class="p-2 bg-white rounded-t-lg">
          <textarea id="shift" v-model="shiftsText" rows="10" class="w-full p-2.5 text-sm text-gray-900 bg-white border-0 focus:ring-0" placeholder="Copy and Paste..." required ></textarea>
        </div>
        <div class="flex items-center justify-between px-3 py-2 border-t">
          <div class="relative flex items-center max-w-[3rem]">
            <input type="number" v-model="nwh" class="bg-white border-2 border-gray-200 rounded-md h-11 text-center text-gray-900 text-sm focus:ring-blue-500 focus:border-blue-500 block w-full py-2.5" placeholder="8" required />
          </div>
          <button type="submit" class="inline-flex items-center py-2.5 px-4 text-xs font-medium text-center text-white bg-blue-600 rounded-lg focus:ring-4 focus:ring-blue-200 hover:bg-blue-700">
            Calculate
          </button>
          <div class="flex ps-0 space-x-1 rtl:space-x-reverse sm:ps-2">
            <button @click="setShowCautionModal" type="button" class="inline-flex justify-center items-center p-2 text-gray-500 rounded cursor-pointer hover:text-gray-900 hover:bg-gray-100">
              <svg class="w-6 h-6" aria-hidden="true" xmlns="http://www.w3.org/2000/svg" width="24" height="24" fill="none" viewBox="0 0 24 24">
                <path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 13V8m0 8h.01M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z"/>
              </svg>
            </button>
            <button @click="setShowDetailModal" type="button" class="inline-flex justify-center items-center p-2 text-gray-500 rounded cursor-pointer hover:text-gray-900 hover:bg-gray-100">
              <svg class="w-6 h-6" aria-hidden="true" xmlns="http://www.w3.org/2000/svg" width="24" height="24" fill="none" viewBox="0 0 24 24">
                <path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 11h2v5m-2 0h4m-2.592-8.5h.01M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z"/>
              </svg>
            </button>
            <button @click="setShowInfoModal" type="button" class="inline-flex justify-center items-center p-2 text-gray-500 rounded cursor-pointer hover:text-gray-900 hover:bg-gray-100">
              <svg class="w-6 h-6" aria-hidden="true" xmlns="http://www.w3.org/2000/svg" width="24" height="24" fill="none" viewBox="0 0 24 24">
                <path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.529 9.988a2.502 2.502 0 1 1 5 .191A2.441 2.441 0 0 1 12 12.582V14m-.01 3.008H12M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z"/>
              </svg>
            </button>
          </div>
        </div>
      </div>
    </form>
    <div class="relative overflow-x-auto shadow-md sm:rounded-lg">
      <table class="w-full text-sm text-center rtl:text-right text-gray-500">
        <thead class="text-xs text-gray-700 uppercase bg-gray-50">
          <tr>
            <th scope="col" class="px-2 py-1">Period</th>
            <th scope="col" class="px-2 py-1">Hours /<br class="block sm:hidden"> </br>Days</th>
            <th scope="col" class="px-2 py-1">Delete</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(res, i) in result" :key="i" class="odd:bg-white even:bg-gray-50 border-b">
            <th scope="row" class="px-2 py-1 font-medium text-gray-900 whitespace-nowrap">{{ res.period }}</th>
            <td class="px-2 py-1">{{ res.hours }} /<br class="block sm:hidden"> </br>{{ res.days }}d</td>
            <td class="px-2 py-1">
              <button @click="dele(i)" class="font-medium inline-block">
                <svg class="w-[20px] h-[20px] text-red-600" aria-hidden="true" xmlns="http://www.w3.org/2000/svg" width="24" height="24" fill="none" viewBox="0 0 24 24">
                  <path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18 17.94 6M18 18 6.06 6"/>
                </svg>
              </button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
  <Modal v-if="showInfoModal" @set-show="setShowInfoModal">
    <template #header>#WorkHolicとは?</template>
    <template #detail>
      <p>このWebアプリは，特定の形式で与えられたシフト表を解析しJSON形式に変換したうえで，半月分の勤務時間を計算します．</p>
      <p>使用方法：シフト表全体をコピぺし，Calculateボタンを押してください．勤務時間に年次有給休暇を考慮したい場合は，全ての日を表示させたシフト表をコピペし，所定労働時間(h)を入力した後，Calculateボタンを押してください．</p>
    </template>
  </Modal>
  <Modal v-if="showDetailModal" @set-show="setShowDetailModal">
    <template #header>#仕様</template>
    <template #detail>
      <p>このサービスを使用できるシフト表の形式は以下の通りです．</p>
      <pre class="border-2 m-3 flex justify-center"><code>
MM/DD(W)
 hh:mm-hh:mm (w)
  [休]  hh:mm-hh:mm (w)
 LABEL
      </code></pre>
      <ul class="list-disc list-inside">
        <li>月日・曜日が上記の形式で記載されている．</li>
        <li>曜日は<code>Su, Mo, Tu, We, Th, Fr, Sa</code>と記載されている．</li>
        <li>勤務がある日は，勤務開始/終了時間と実働時間が上記の形式で記載されている．</li>
        <li>休憩時間がある場合は，<code>[休]</code>に続いて休憩開始/終了時間と休憩時間が記載されている．</li>
        <li>勤務がある日は，ラベル位置に勤務名が記載されている．</li>
        <li>勤務がない日は，ラベル位置に<code>----</code>もしくは<code>休日</code>と記載されている．</li>
        <li>特別有給休暇を使用する日は，ラベル位置に<code>休暇</code>と記載されている．</li>
        <li>欠勤した日は，ラベル位置に<code>欠勤</code>が含まれている．</li>
        <li>同日に複数の勤務があっても良い．</li>
      </ul>
      <p>計算された合計の勤務時間には休憩時間や欠勤した勤務は含まれません．ただし特別有給休暇を使用した日がある場合，所定労働時間分の勤務時間が含まれます．</p>
    </template>
  </Modal>
  <Modal v-if="showCautionModal" @set-show="setShowCautionModal">
    <template #header>#ご利用にあたって（利用規約）</template>
    <template #detail>
      <p>以下の利用規約は，このウェブサイト上で提供するサービス（以下，本サービス）の利用条件を本サービスの作成者（以下，作成者）が定めるものです．ユーザーの皆様（以下，ユーザー）には，本規約に従って，本サービスをご利用いただきます．</p>
      <p>本規約は，ユーザーと作成者との間の本サービスの利用に関わる一切の関係に適用されるものとします．</p>
      <p>ユーザーは本サービスの利用にあたり，次に示す行為をしてはなりません．法令または公序良俗に違反する行為・犯罪行為に関連する行為・本サービスの内容等，本サービスに含まれる著作権，商標権ほか知的財産権を侵害する行為・作成者，ほかのユーザー，またはその他第三者のサーバーまたはネットワークの機能を破壊したり，妨害したりする行為・本サービスによって得られた情報を商業的に利用する行為・サービスの運営を妨害するおそれのある行為・不正アクセスをし，またはこれを試みる行為・他のユーザーに関する個人情報等を収集または蓄積する行為・不正な目的を持って本サービスを利用する行為・本サービスの他のユーザーまたはその他の第三者に不利益，損害，不快感を与える行為・作成者が許諾しない本サービス上での宣伝，広告，勧誘，または営業行為・作成者のサービスに関連して，反社会的勢力に対して直接または間接に利益を供与する行為・その他，作成者が不適切と判断する行為</p>
      <p>本サービスはユーザーに事前に通知することなく本サービスの全部または一部の提供を停止または中断，内容を変更することができるものとします．これによってユーザーまたは第三者に生じたいかなる損害または不利益について一切の責任を負わないものとします．</p>
      <p>作成者は，本サービスに事実上または法律上の瑕疵（安全性，信頼性，正確性，完全性，有効性，特定の目的への適合性，セキュリティなどに関する欠陥，エラーやバグ，権利侵害などを含みます．）がないことを明示的にも黙示的にも保証しておりません．作成者は，本サービスに起因してユーザーに生じたあらゆる損害について，作成者の故意又は重過失による場合を除き，一切の責任を負いません．</p>
      <p>作成者はユーザーの個別の同意を要せず，本規約を変更することができるものとします．</p>
      <p>本サービスは，特定の人物，団体，企業等と関与を持つものではありません．</p>
      <p>本サービスではGoogle Analyticsを利用しています．Google社によってサイトの利用状況が収集・分析・記録され，作成者はその結果を把握しさらなるサービス向上に努めています．Google Analyticsにより得られる情報には，個人を特定する情報は一切含まれません．詳しい情報については，Google社のプライバシーポリシー及びGoogle Analyticsのサイトをご覧ください．</p>
      <p>Google Analytics及びサービスの一時記録のため，Cookieを使用しています．</p>
    </template>
  </Modal>
</template>

<style>
li code {
  background-color: rgb(229, 240, 249);
  font-size: smaller;
  padding: 2px 4px;
  margin: 0 2px;
}
</style>